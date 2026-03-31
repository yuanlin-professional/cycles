# motion_triangle.h - 运动模糊三角形顶点与法线插值

## 概述
本文件实现了运动模糊(motion blur)三角形图元的顶点位置和法线插值功能。运动三角形以常规三角形形式存储，附加额外的顶点位置和法线数据作为 `ATTR_STD_MOTION_VERTEX_POSITION` 和 `ATTR_STD_MOTION_VERTEX_NORMAL` 网格属性。通过在两个相邻时间步之间插值，计算给定光线时间处的三角形顶点坐标和平滑法线。

## 核心函数

### motion_triangle_verts_for_step()
- **签名**: `ccl_device_inline void motion_triangle_verts_for_step(KernelGlobals kg, const uint3 tri_vindex, int offset, const int numverts, const int numsteps, int step, float3 verts[3])`
- **功能**: 获取指定时间步的三角形三个顶点位置。中心步从 `tri_verts` 数组读取，其他步从 `attributes_float3` 运动属性数组中读取。

### motion_triangle_normals_for_step()
- **签名**: `ccl_device_inline void motion_triangle_normals_for_step(KernelGlobals kg, const uint3 tri_vindex, int offset, const int numverts, const int numsteps, int step, float3 normals[3])`
- **功能**: 获取指定时间步的三角形三个顶点法线。中心步从 `tri_vnormal` 数组读取，其他步从运动属性数组中读取。

### motion_triangle_compute_info()
- **签名**: `ccl_device_inline void motion_triangle_compute_info(KernelGlobals kg, const int object, const float time, const int prim, uint3 *tri_vindex, int *numsteps, int *step, float *t)`
- **功能**: 计算运动三角形的插值信息：时间步数、当前步索引、插值因子和三角形顶点索引。是其他运动三角形函数的共用信息准备步骤。

### motion_triangle_vertices()
- **签名**: `ccl_device_inline void motion_triangle_vertices(KernelGlobals kg, const int object, const int prim, const float time, float3 verts[3])`
- **功能**: 计算给定时间的运动三角形三个顶点位置。提供两个重载版本：一个接受完整参数（用于内部优化），一个仅需 object、prim 和 time（便捷接口）。

### motion_triangle_normals()
- **签名**: `ccl_device_inline void motion_triangle_normals(KernelGlobals kg, const int object, const uint3 tri_vindex, const int numsteps, const int numverts, const int step, const float t, float3 normals[3])`
- **功能**: 计算给定时间的运动三角形三个顶点法线，在两个时间步之间插值后归一化。

### motion_triangle_vertices_and_normals()
- **签名**: `ccl_device_inline void motion_triangle_vertices_and_normals(KernelGlobals kg, const int object, const int prim, const float time, float3 verts[3], float3 normals[3])`
- **功能**: 同时计算运动三角形的顶点位置和法线，共享运动信息的计算以减少冗余。

### motion_triangle_smooth_normal()
- **签名**: `ccl_device_inline float3 motion_triangle_smooth_normal(KernelGlobals kg, const float3 Ng, const int object, const int prim, const float u, float v, const float time)`
- **功能**: 计算运动三角形在交点处的平滑法线。使用重心坐标(u, v)在三个顶点法线之间插值。提供三个重载版本，其中一个额外输出 x/y 方向偏移位置处的法线(用于凹凸贴图/bump mapping)。

## 依赖关系
- **内部头文件**: `kernel/bvh/util.h`, `kernel/geom/triangle.h`
- **被引用**: `geom/motion_triangle_intersect.h`, `geom/motion_triangle_shader.h`, `svm/wireframe.h`, `svm/tex_coord.h`, `svm/bevel.h`, `osl/services_gpu.h`, `osl/services_shared.h`, `light/triangle.h`, `integrator/shade_surface.h`, `integrator/mnee.h`

## 实现细节 / 关键算法

### 顶点插值
- 与运动曲线相同的时间步映射：`maxstep = numsteps * 2`，中心步存储在常规数组中
- 插值公式: `verts[i] = (1.0 - t) * verts_step[i] + t * verts_step_next[i]`

### 法线插值
- 法线在插值后需要**归一化**以保持单位长度: `normals[i] = normalize((1-t) * n_step[i] + t * n_next[i])`
- 平滑法线通过重心坐标插值: `N = safe_normalize(w * n0 + u * n1 + v * n2)`，若结果为零向量则回退到几何法线 Ng

### 凹凸贴图支持
带微分的 `motion_triangle_smooth_normal()` 重载计算三个法线：
- 中心法线 N(u, v)
- x 偏移法线 N_x(u + du.dx, v + dv.dx)
- y 偏移法线 N_y(u + du.dy, v + dv.dy)

## 关联文件
- `kernel/geom/triangle.h` - 提供 `triangle_interpolate()` 基础插值函数
- `kernel/geom/motion_triangle_intersect.h` - 运动三角形相交测试
- `kernel/geom/motion_triangle_shader.h` - 运动三角形着色设置
