# curve_intersect.h - 曲线图元光线相交测试

## 概述
本文件实现了 Cycles 渲染器中曲线图元与光线的相交测试算法。代码改编自 Intel Embree 的 `curve_intersector_sweep.h`，以确保 Embree CPU 光线追踪与 Cycles GPU 光线追踪之间的精确匹配。支持三种曲线类型：Catmull-Rom 粗曲线(thick curve)、飘带曲线(ribbon)和线性粗曲线(thick linear)。

## 核心函数

### catmull_rom_basis_eval() / catmull_rom_basis_derivative() / catmull_rom_basis_derivative2()
- **签名**: `ccl_device_inline float4 catmull_rom_basis_eval(const float4 curve[4], float u)`
- **功能**: Catmull-Rom 样条曲线求值函数族。分别计算曲线在参数 u 处的位置、一阶导数和二阶导数。使用四个控制点(curve[0..3])进行基函数评估。

### cylinder_intersect()
- **签名**: `ccl_device_inline bool cylinder_intersect(const float3 cylinder_start, const float3 cylinder_end, const float cylinder_radius, const float3 ray_D, float2 *t_o, ...)`
- **功能**: 光线与圆柱体的相交测试。通过求解二次方程计算光线与有限圆柱体的两个交点参数 t、对应的 u 参数值和法线方向。处理了平行光线的特殊情况。

### curve_intersect_iterative()
- **签名**: `ccl_device bool curve_intersect_iterative(const float3 ray_D, const float ray_tmin, float *ray_tmax, const float dt, const float4 curve[4], float u, float t, const bool use_backfacing, Intersection *isect)`
- **功能**: 使用牛顿法(Newton-Raphson)迭代求解精确的曲线-光线交点。在 Bezier 细分定位近似交点后，通过雅可比(Jacobian)迭代收敛到真实交点。最多执行 `CURVE_NUM_JACOBIAN_ITERATIONS`(5) 次迭代。

### curve_intersect_recursive()
- **签名**: `ccl_device bool curve_intersect_recursive(const float3 ray_P, const float3 ray_D, const float ray_tmin, float ray_tmax, float4 curve[4], Intersection *isect)`
- **功能**: 粗曲线(thick curve)的递归相交测试主入口。使用 Bezier 细分与圆柱体包围层次结构逐步缩小搜索范围，在稳定区域直接进行迭代求解，在不稳定区域进一步细分。

### ribbon_intersect()
- **签名**: `ccl_device_inline bool ribbon_intersect(const float3 ray_org, const float3 ray_D, const float ray_tmin, float ray_tmax, const int N, float4 curve[4], Intersection *isect)`
- **功能**: 飘带曲线(ribbon)相交测试。将曲线离散为 N 段，每段构建四边形(quad)，通过光线空间变换后进行光线-四边形相交测试。

### cone_sphere_intersect()
- **签名**: `ccl_device_inline bool cone_sphere_intersect(const float4 curve[4], const float3 ray_D, float *t_o, float *u_o, float3 *Ng_o)`
- **功能**: 线性粗曲线的锥体-球体相交测试。将线性曲线段近似为截锥体，端部用球体封盖，求解光线与这些几何体的交点。

### linear_curve_intersect()
- **签名**: `ccl_device bool linear_curve_intersect(const float3 ray_P, const float3 ray_D, const float ray_tmin, float ray_tmax, float4 curve[4], Intersection *isect)`
- **功能**: 线性粗曲线的相交测试入口。先将曲线段平移到光线附近以提高数值稳定性，然后调用 `cone_sphere_intersect()` 进行测试。

### curve_intersect()
- **签名**: `ccl_device_forceinline bool curve_intersect(KernelGlobals kg, Intersection *isect, const float3 ray_P, const float3 ray_D, const float tmin, const float tmax, const int object, const int prim, const float time, const int type)`
- **功能**: 统一的曲线相交入口函数。根据曲线类型(RIBBON/THICK/THICK_LINEAR)分发到对应的相交算法。处理运动模糊情况下的曲线关键帧插值。

### curve_shader_setup()
- **签名**: `ccl_device_inline void curve_shader_setup(KernelGlobals kg, ShaderData *sd, float3 P, float3 D, float t, const int isect_prim)`
- **功能**: 在曲线交点处设置着色数据(ShaderData)。计算交点的位置、法线(N)、几何法线(Ng)、切线导数(dPdu/dPdv)等。对飘带曲线使用圆化平滑法线，对粗曲线使用从曲线中心到交点的方向作为法线。

## 依赖关系
- **内部头文件**: `kernel/geom/motion_curve.h`, `kernel/geom/object.h`
- **被引用**: `geom/shader_data.h`, `bvh/bvh.h`

## 实现细节 / 关键算法

### Bezier 递归细分算法
粗曲线相交使用多层次的方法：
1. 将 Catmull-Rom 曲线在参数范围内均匀分为 `CURVE_NUM_BEZIER_STEPS`(2) 段
2. 对每段构建内外包围圆柱体进行快速剔除
3. 通过半平面(half-plane)裁剪进一步收紧参数范围
4. 若圆柱体测试通过但精度不足，递归细分（最大深度 `CURVE_NUM_BEZIER_SUBDIVISIONS`=3 或不稳定区域 4 层）
5. 达到终止深度后调用 `curve_intersect_iterative()` 进行牛顿迭代精确求解

### 数值稳定性措施
- 在递归和线性相交中，先将光线原点平移到曲线中心附近(通过计算 dt 偏移)
- 牛顿迭代中跟踪误差项(P_err, Q_err, R_err, f_err, g_err)，基于浮点精度(16*FLT_EPSILON)设定收敛阈值
- 圆柱体半径使用 one_plus_ulp/one_minus_ulp 因子进行保守扩展/收缩

### 飘带曲线(Ribbon)
- 将控制点变换到光线空间(ray space)后，在 2D 空间中进行圆柱体剔除和四边形相交
- 使用法线加宽(wn)构造四边形的上下边界，模拟曲线宽度
- 包含自相交避免因子(avoidance_factor)

## 关联文件
- `kernel/geom/motion_curve.h` - 运动模糊曲线控制点插值
- `kernel/geom/object.h` - 对象变换操作
- `kernel/geom/curve.h` - 曲线着色属性读取
- `kernel/bvh/bvh.h` - 层次包围体(BVH)遍历调用本文件的相交函数
