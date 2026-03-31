# triangle.h - 三角形光源(网格发光体)采样

## 概述
本文件实现了 Cycles 渲染器中三角形光源（网格发光体/发光面片）的采样和 PDF 计算。三角形光源是场景中带有发射着色器的网格三角形。采样策略根据着色点到三角形平面的距离与三角形边长的比值自适应选择：近距离使用基于球面三角形的立体角采样（精确但昂贵），远距离使用面积采样（简单但对大三角形有偏差）。

## 核心函数

### triangle_world_space_vertices()
- **签名**: `ccl_device_inline bool triangle_world_space_vertices(KernelGlobals kg, const int object, const int prim, const float time, float3 V[3])`
- **功能**: 获取三角形在世界空间中的顶点坐标。支持运动模糊（通过 `motion_triangle_vertices` 获取指定时间的顶点）和实例化变换（通过对象变换矩阵将局部坐标转换到世界空间）。

### triangle_light_pdf_area_sampling()
- **签名**: `ccl_device_inline float triangle_light_pdf_area_sampling(const float3 Ng, const float3 I, const float t)`
- **功能**: 计算三角形面积采样的 PDF（面积到立体角的转换）。与 `light_pdf_area_to_solid_angle` 类似，但使用 `fabsf(dot(Ng, I))` 支持双面发射。

### triangle_light_pdf()
- **签名**: `ccl_device_forceinline float triangle_light_pdf(KernelGlobals kg, const ccl_private ShaderData *sd, const float t)`
- **功能**: 计算三角形光源的采样 PDF。根据距离到平面与最长边的比值自适应选择算法：当 `longest_edge^2 > distance_to_plane^2` 时使用精确的立体角 PDF（`1/solid_angle`），否则使用面积采样 PDF。在未使用光源树时，额外乘以分布 PDF 因子。

### triangle_light_sample()
- **签名**: `template<bool in_volume_segment> ccl_device_forceinline bool triangle_light_sample(KernelGlobals kg, const int prim, const int object, const float2 rand, const float time, ccl_private LightSample *ls, const float3 P)`
- **功能**: 在三角形光源上采样一个点。自适应选择采样策略：
  - **立体角采样**: 基于 James Arvo 论文"Stratified Sampling of Spherical Triangles"的改进版本。将三角形投影到单位球面，计算球面三角形的面积（即立体角），然后在球面三角形上均匀采样后反投影到平面三角形。
  - **面积采样**: 使用 Eric Heitz 的"A Low-Distortion Map Between Triangle and Square"方法在三角形上均匀采样。

### triangle_light_valid_ray_segment()
- **签名**: `ccl_device_inline bool triangle_light_valid_ray_segment(KernelGlobals kg, const float3 P, const float3 D, ccl_private Interval<float> *t_range, const ccl_private LightSample *ls)`
- **功能**: 计算光线被三角形光源照射的有效线段范围。双面发射时整个线段有效，单面发射时与三角形平面求交确定有效半空间。

### triangle_light_tree_parameters()
- **签名**: `template<bool in_volume_segment> ccl_device_forceinline bool triangle_light_tree_parameters(KernelGlobals kg, const ccl_global KernelLightTreeEmitter *kemitter, const float3 centroid, const float3 P, const float3 N, const KernelBoundingCone bcone, ccl_private float &cos_theta_u, ccl_private float2 &distance, ccl_private float3 &point_to_centroid)`
- **功能**: 为光源树重要性计算提供三角形光源的几何参数。通过遍历三个顶点计算最小包围角和最大距离。可见性由正面朝向和表面上方两个条件共同决定。

## 依赖关系
- **内部头文件**: `kernel/globals.h`, `kernel/light/common.h`, `kernel/geom/motion_triangle.h`, `kernel/geom/object.h`, `kernel/geom/triangle.h`, `util/math_fast.h`, `util/math_intersect.h`
- **被引用**: `kernel/light/light.h`, `kernel/light/tree.h`

## 实现细节 / 关键算法
1. **自适应采样策略**: 将着色点到三角形平面的距离与最长边长比较。当距离较小（近距离观察大三角形）时，面积采样产生的方差很大，因此切换到立体角采样。立体角采样的 PDF 在立体角空间中均匀，不受三角形面积和距离的影响。
2. **球面三角形采样(Arvo)**: 算法步骤：(1) 将三角形顶点投影到单位球面得到 A, B, C；(2) 计算球面三角形面积（即立体角）`2*atan2(|A.(BxC)|, 1+B.C+A.C+A.B)`；(3) 按面积比例随机选择子区域，计算第三顶点 C'；(4) 在 C'-B 弧上均匀采样方向；(5) 与平面三角形求交得到最终采样点。
3. **低失真三角形-正方形映射(Heitz)**: 面积采样使用 Eric Heitz 的方法将 [0,1]^2 映射到三角形，比传统的重心坐标映射具有更低的失真度，产生更均匀的采样分布。
4. **发射方向检查**: 通过着色器的 `SD_MIS_FRONT` 和 `SD_MIS_BACK` 标志判断三角形的发射方向。单面发射的三角形在背面不产生采样，双面发射时使用 `fabsf` 处理法线点积。
5. **运动模糊处理**: CDF 基于固定时间 (time=-1) 的三角形面积构建，但实际采样使用当前帧时间的顶点位置。当面积因运动模糊改变时，需要用固定时间的面积重新计算分布 PDF。

## 关联文件
- `kernel/light/common.h` - 提供 `LightSample` 结构体
- `kernel/light/light.h` - 在 `light_sample` 中调用 `triangle_light_sample`
- `kernel/light/tree.h` - 调用 `triangle_light_tree_parameters` 计算光源树重要性
- `kernel/geom/triangle.h` - 提供 `triangle_vertices` 等基础几何函数
- `kernel/geom/motion_triangle.h` - 提供运动模糊三角形的顶点获取
