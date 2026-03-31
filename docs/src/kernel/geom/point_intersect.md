# point_intersect.h - 点云图元光线相交测试

## 概述
本文件实现了 Cycles 渲染器中点云(Point Cloud)图元与光线的相交测试算法。将点云中的每个点视为球体，通过解析求解光线-球体相交方程来检测命中。同时提供了交点处的着色数据设置功能。

## 核心函数

### point_intersect_test()
- **签名**: `ccl_device_forceinline bool point_intersect_test(const float4 point, const float3 ray_P, const float3 ray_D, const float ray_tmin, const float ray_tmax, float *t)`
- **功能**: 核心的光线-球体相交测试。将点视为以 `point.xyz` 为中心、`point.w` 为半径的球体。通过以下步骤求解：
  1. 计算光线到球心的投影距离 `projC0`
  2. 计算垂直距离的平方 `l2`
  3. 若 `l2 > r2` 则光线未命中
  4. 计算半球内的偏移 `td`，得到前交点 `t_front = projC0 - td`
  5. 当前实现仅使用前面相交(背面剔除 always on)

### point_intersect()
- **签名**: `ccl_device_forceinline bool point_intersect(KernelGlobals kg, Intersection *isect, const float3 ray_P, const float3 ray_D, const float ray_tmin, const float ray_tmax, const int object, const int prim, const float time, const int type)`
- **功能**: 点云图元的完整相交入口。处理运动模糊(通过 `motion_point()` 获取时间插值后的位置和半径)和静态点。调用 `point_intersect_test()` 执行几何测试，命中时填充 Intersection 结构(u=0, v=0 因点没有表面参数化)。

### point_shader_setup()
- **签名**: `ccl_device_inline void point_shader_setup(KernelGlobals kg, ShaderData *sd, const Intersection *isect, const Ray *ray)`
- **功能**: 在点云交点处设置着色数据。计算内容：
  - **位置**: `P = ray_P + ray_D * t`
  - **法线**: 从点中心到交点的方向 `N = normalize(P - center)`
  - **着色器**: 从 `points_shader` 数组读取
  - **UV**: 设为 isect 中的 u, v 值(目前为零)
  - **dPdu/dPdv**: 设为零(点没有表面参数化)

## 依赖关系
- **内部头文件**: `kernel/globals.h`, `kernel/types.h`, `kernel/geom/motion_point.h`, `kernel/geom/object.h`
- **被引用**: `geom/shader_data.h`, `bvh/bvh.h`

## 实现细节 / 关键算法

### 光线-球体相交
采用经典的解析求解方法：
1. 计算光线原点到球心的向量 c0
2. 将 c0 投影到光线方向得到最近点参数 `projC0 = dot(c0, D) / dot(D, D)`
3. 计算垂直距离平方 `l2 = |c0|^2 - projC0^2 * |D|^2`（这里实际用了 `perp = c0 - projC0 * D`）
4. 若 `l2 <= r2`，交点参数为 `t = projC0 +/- sqrt((r2 - l2) * rd2)`

### 背面剔除
当前实现**始终启用背面剔除**，仅返回前交点(t_front)。代码中有后交点(t_back)的注释代码，使用 `#if 0` 禁用。

### 法线计算
点的法线方向从球心指向交点: `Ng = normalize(P - center)`。这与标准球体的外向法线一致。运动模糊场景中球心位置通过 `motion_point()` 获取。

整个文件被 `__POINTCLOUD__` 预处理宏保护。

## 关联文件
- `kernel/geom/motion_point.h` - 运动模糊点位置插值
- `kernel/geom/object.h` - 对象变换(点中心的世界空间位置)
- `kernel/geom/shader_data.h` - 着色数据初始化中调用 `point_shader_setup()`
- `kernel/bvh/bvh.h` - BVH 遍历中调用 `point_intersect()`
