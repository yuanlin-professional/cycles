# motion_triangle_intersect.h - 运动模糊三角形光线相交测试

## 概述
本文件实现了运动模糊(motion blur)三角形图元与光线的相交测试。核心策略是先根据光线时间插值计算出三角形在当前时刻的顶点位置，然后使用标准的光线-三角形相交算法进行测试。同时提供了局部相交(local intersection)版本，用于次表面散射等需要在同一对象内检测多个命中的场景。

## 核心函数

### motion_triangle_point_from_uv()
- **签名**: `ccl_device_inline float3 motion_triangle_point_from_uv(KernelGlobals kg, ShaderData *sd, const float u, const float v, const float3 verts[3])`
- **功能**: 使用重心坐标(u, v)在运动三角形的三个顶点之间插值计算交点的世界空间位置。公式: `P = v0 + u * (v1 - v0) + v * (v2 - v0)`。若对象变换尚未应用则执行对象到世界空间的变换。

### motion_triangle_intersect()
- **签名**: `ccl_device_inline bool motion_triangle_intersect(KernelGlobals kg, Intersection *isect, const float3 P, const float3 dir, const float tmin, const float tmax, const float time, const uint visibility, const int object, const int prim, const int prim_addr)`
- **功能**: 运动三角形的标准光线相交测试。先通过 `motion_triangle_vertices()` 获取当前时刻的三个顶点位置，再调用 `ray_triangle_intersect()` 执行未优化的光线-三角形相交。通过可见性标志(visibility flag)进行额外过滤。命中类型设为 `PRIMITIVE_MOTION_TRIANGLE`。

### motion_triangle_intersect_local()
- **签名**: `ccl_device_inline bool motion_triangle_intersect_local(KernelGlobals kg, LocalIntersection *local_isect, const float3 P, const float3 dir, const float time, const int object, const int prim, const float tmin, const float tmax, uint *lcg_state, const int max_hits)`
- **功能**: 运动三角形的局部相交测试（用于次表面散射）。仅与同一对象内的图元相交，支持多命中记录和蓄水池抽样。当 `max_hits == 0` 时仅检测是否有命中(不记录详细信息)。记录交点信息和几何法线 `Ng`。

## 依赖关系
- **内部头文件**: `kernel/globals.h`, `kernel/types.h`, `kernel/geom/geom_intersect.h`, `kernel/geom/motion_triangle.h`, `kernel/geom/object.h`, `util/math_intersect.h`
- **被引用**: `geom/motion_triangle_shader.h`, `bvh/bvh.h`

## 实现细节 / 关键算法

### 相交策略
运动三角形采用"即时插值"策略：在相交测试时实时计算当前时刻的顶点位置，而非预计算。这种方法虽然计算开销较大（每次测试都需要顶点插值），但避免了存储所有时刻三角形数据的内存开销。

### 可见性过滤
标准相交版本在确认几何相交后，还会检查 `prim_visibility` 与 `visibility` 掩码的与运算结果。可见性测试在此处执行（而非更早），因为假设大部分三角形已被 BVH 节点标志剔除。

### 局部相交
- 调用 `local_intersect_get_record_index()` 获取命中记录索引
- 返回值始终为 false（表示不终止遍历），以继续收集更多命中
- 局部相交受 `__BVH_LOCAL__` 预处理宏保护

## 关联文件
- `kernel/geom/motion_triangle.h` - 运动三角形顶点插值
- `kernel/geom/geom_intersect.h` - 局部相交记录管理
- `kernel/geom/object.h` - 对象变换
- `kernel/bvh/bvh.h` - 层次包围体(BVH)遍历中调用本文件的相交函数
- `util/math_intersect.h` - 提供 `ray_triangle_intersect()` 基础算法
