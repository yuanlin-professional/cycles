# triangle_intersect.h - 三角形光线相交测试

## 概述
本文件实现了 Cycles 渲染器中三角形图元与光线的相交测试算法。提供标准相交测试和局部相交测试(用于次表面散射)两种模式。同时包含了通过重心坐标计算交点位置的工具函数和三角形着色数据设置函数。为了加速层次包围体(BVH)光线相交，使用预计算的三角形存储以换取更多内存使用。

## 核心函数

### triangle_intersect()
- **签名**: `ccl_device_inline bool triangle_intersect(KernelGlobals kg, Intersection *isect, const float3 P, const float3 dir, const float tmin, const float tmax, const uint visibility, const int object, const int prim, const int prim_addr)`
- **功能**: 标准的光线-三角形相交测试。从 `tri_vindex` 获取顶点索引、从 `tri_verts` 获取顶点位置，调用 `ray_triangle_intersect()` 执行核心相交算法。通过 `prim_visibility` 进行可见性标志过滤。命中时将图元类型设为 `PRIMITIVE_TRIANGLE`。

### triangle_intersect_local()
- **签名**: `ccl_device_inline bool triangle_intersect_local(KernelGlobals kg, LocalIntersection *local_isect, const float3 P, const float3 dir, const int object, const int prim, const float tmin, const float tmax, uint *lcg_state, const int max_hits)`
- **功能**: 局部相交测试，用于次表面散射(subsurface scattering)场景。仅与同一对象内的图元相交，支持多命中记录。通过 `local_intersect_get_record_index()` 管理命中记录(支持蓄水池抽样)。当 `max_hits == 0` 时仅检测是否命中。同时记录几何法线 `Ng`。始终返回 false(不终止遍历)。

### triangle_point_from_uv()
- **签名**: `ccl_device_inline float3 triangle_point_from_uv(KernelGlobals kg, ShaderData *sd, const int isect_prim, const float u, const float v)`
- **功能**: 使用重心坐标计算三角形上的世界空间位置。公式: `P = a + u * (b - a) + v * (c - a)`(比 `(1-u-v)*a + u*b + v*c` 精度更好)。若变换尚未应用则执行对象到世界空间变换。

### triangle_shader_setup()
- **签名**: `ccl_device_inline void triangle_shader_setup(KernelGlobals kg, ShaderData *sd)`
- **功能**: 在三角形交点处设置着色数据。执行步骤：
  1. 从 `tri_shader` 读取着色器 ID
  2. 通过 `triangle_point_from_uv()` 计算精确位置
  3. 通过 `triangle_normal()` 计算面法线 Ng
  4. 若着色器启用了 `SHADER_SMOOTH_NORMAL`，计算平滑法线 N
  5. 通过 `triangle_dPdudv()` 计算 dPdu/dPdv

## 依赖关系
- **内部头文件**: `kernel/globals.h`, `kernel/geom/geom_intersect.h`, `kernel/geom/object.h`, `kernel/geom/triangle.h`, `util/math_float3.h`, `util/math_intersect.h`
- **被引用**: `geom/shader_data.h`, `svm/bevel.h`, `bvh/bvh.h`

## 实现细节 / 关键算法

### 相交算法
核心相交计算委托给 `ray_triangle_intersect()`(定义在 `util/math_intersect.h` 中)，这是一个经过优化的光线-三角形相交测试。本文件负责从内核数据中提取顶点数据并处理结果。

### 可见性过滤
相交成功后检查 `prim_visibility[prim_addr] & visibility`。可见性测试在此处(而非 BVH 节点级别)执行，基于假设大部分不可见三角形已被 BVH 节点标志剔除。受 `__VISIBILITY_FLAG__` 保护。

### 局部相交与蓄水池抽样
局部相交通过 `local_intersect_get_record_index()` (来自 `geom_intersect.h`) 管理命中记录。在多命中模式下，当命中数超过 `max_hits` 时使用蓄水池抽样随机替换，确保公平采样。受 `__BVH_LOCAL__` 保护。

### 位置精度
`triangle_point_from_uv()` 使用 `P = a + u*(b-a) + v*(c-a)` 而非 `(1-u-v)*a + u*b + v*c`，注释说明前者给出略好的精度(减少中间值的取消误差)。

## 关联文件
- `kernel/geom/triangle.h` - 三角形法线、平滑法线、dPdu/dPdv 等
- `kernel/geom/geom_intersect.h` - 局部相交记录管理
- `kernel/geom/object.h` - 对象变换
- `kernel/geom/shader_data.h` - 着色数据初始化调用 `triangle_shader_setup()`
- `kernel/bvh/bvh.h` - BVH 遍历中调用 `triangle_intersect()`
- `util/math_intersect.h` - 核心光线-三角形相交算法
