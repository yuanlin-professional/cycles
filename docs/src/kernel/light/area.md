# area.h - 面积光源采样与求交

## 概述
本文件实现了 Cycles 渲染器中面积光源(矩形光和椭圆光)的采样、求交和概率密度函数(PDF)计算。核心采样算法基于 Carlos Urena 等人的论文"An Area-Preserving Parametrization for Spherical Rectangles"，通过球面矩形参数化实现高效的重要性采样。同时支持光源扩散角(spread angle)的衰减模拟，用于模拟柔光箱栅格效果。

## 核心函数

### area_light_rect_sample()
- **签名**: `ccl_device_inline float area_light_rect_sample(const float3 P, ccl_private float3 *light_p, const float3 axis_u, const float len_u, const float3 axis_v, const float len_v, const float2 rand, bool sample_coord)`
- **功能**: 对球面矩形进行重要性采样。根据着色点 P 和矩形光源参数，计算立体角下的 PDF，并可选地采样光源表面上的新位置。当立体角过小时回退到平面采样 PDF 以避免数值精度问题。

### area_light_spread_attenuation()
- **签名**: `ccl_device float area_light_spread_attenuation(const float3 D, const float3 lightNg, const float tan_half_spread, const float normalize_spread)`
- **功能**: 模拟柔光箱栅格的光线衰减效果。根据光线方向与光源法线的夹角以及扩散半角计算衰减系数。当扩散角为 0 时，仅在法线方向有贡献。

### area_light_spread_clamp_light()
- **签名**: `ccl_device bool area_light_spread_clamp_light(const float3 P, const float3 lightNg, ccl_private float3 *lightP, ccl_private float3 *axis_u, ccl_private float *len_u, ccl_private float3 *axis_v, ccl_private float *len_v, const float tan_half_spread, ccl_private bool *sample_rectangle)`
- **功能**: 计算覆盖有效采样区域的最小矩形、圆形或椭圆形，以在低扩散角情况下减少噪声。通过裁剪采样区域到光源实际照射范围来提升采样效率。

### area_light_eval()
- **签名**: `template<bool in_volume_segment> ccl_device_inline bool area_light_eval(const ccl_global KernelLight *klight, const float3 ray_P, ccl_private float3 *light_P, ccl_private LightSample *ccl_restrict ls, const float2 rand, bool sample_coord)`
- **功能**: 面积光源的统一求值接口。计算 `eval_fac`（辐射亮度因子）和 `pdf`（概率密度），支持体积段模式和表面模式。在体积段中使用面积采样，在表面模式中根据光源形状选择球面矩形采样或椭圆采样。

### area_light_sample()
- **签名**: `template<bool in_volume_segment> ccl_device_inline bool area_light_sample(const ccl_global KernelLight *klight, const float2 rand, const float3 P, ccl_private LightSample *ls)`
- **功能**: 在面积光源上采样一个点。首先检查着色点是否位于光源正面，然后调用 `area_light_eval` 执行实际采样，最后计算重心坐标(兼容 Embree 和 OptiX 表示法)。

### area_light_intersect()
- **签名**: `ccl_device_inline bool area_light_intersect(const ccl_global KernelLight *klight, const ccl_private Ray *ccl_restrict ray, ccl_private float *t, ccl_private float *u, ccl_private float *v)`
- **功能**: 光线与面积光源的求交测试。支持矩形和椭圆形状，使用 `ray_quad_intersect` 实现。仅在光线从光源正面射入时返回有效交点。

### area_light_sample_from_intersection()
- **签名**: `ccl_device_inline bool area_light_sample_from_intersection(const ccl_global KernelLight *klight, const ccl_private Intersection *ccl_restrict isect, const float3 ray_P, const float3 ray_D, ccl_private LightSample *ccl_restrict ls)`
- **功能**: 从已有的光线求交结果构建光源采样数据，用于前向路径追踪中的多重重要性采样(MIS)权重计算。

### area_light_mnee_sample_update()
- **签名**: `ccl_device_forceinline void area_light_mnee_sample_update(const ccl_global KernelLight *klight, ccl_private LightSample *ls, const float3 P)`
- **功能**: 为 MNEE（多项式牛顿显式估计）更新面积光源采样。对于零扩散角的光源保持方向固定，否则保持位置固定并重新计算 PDF。

### area_light_max_extent()
- **签名**: `ccl_device_forceinline float area_light_max_extent(const ccl_global KernelAreaLight *light)`
- **功能**: 返回光源中心到边界的最大距离，椭圆取两轴最大值的一半，矩形取对角线一半。

### area_light_valid_ray_segment()
- **签名**: `ccl_device_inline bool area_light_valid_ray_segment(const ccl_global KernelAreaLight *light, float3 P, float3 D, ccl_private Interval<float> *t_range)`
- **功能**: 计算光线被面积光源照射的有效线段范围。对于零扩散角使用包围盒/圆柱体求交，否则使用锥体求交进行保守估计。用于体积渲染中的光源贡献范围计算。

### area_light_tree_parameters()
- **签名**: `template<bool in_volume_segment> ccl_device_forceinline bool area_light_tree_parameters(const ccl_global KernelLight *klight, const float3 centroid, const float3 P, const float3 N, const float3 bcone_axis, ccl_private float &cos_theta_u, ccl_private float2 &distance, ccl_private float3 &point_to_centroid)`
- **功能**: 为光源树(light tree)的重要性计算提供面积光源的几何参数，包括张角余弦、距离和方向等。通过遍历四个角点计算最小包围角。

## 依赖关系
- **内部头文件**: `kernel/light/common.h`, `util/math_intersect.h`
- **被引用**: `kernel/light/background.h`, `kernel/light/light.h`, `kernel/light/tree.h`

## 实现细节 / 关键算法
1. **球面矩形采样**: 基于 Urena 论文的算法，将矩形光源投影到着色点的球面上，在立体角空间中均匀采样。使用 `asin` 代替 `acos` 计算内角以避免极小矩形时的数值抵消误差。
2. **扩散角裁剪**: `area_light_spread_clamp_light` 通过计算光源平面上实际有效的照射区域（圆与矩形/椭圆的交集），选择面积最小的采样形状（矩形、圆、椭圆），从而减少无效采样导致的噪声。
3. **椭圆/矩形区分**: 通过 `invarea` 的符号区分光源形状——正值为矩形，负值为椭圆，绝对值为逆面积。
4. **体积段特殊处理**: 在体积渲染中，面积光源使用简单的面积采样而非立体角采样，因为体积中的采样点不在固定表面上。

## 关联文件
- `kernel/light/common.h` - 提供 `LightSample` 结构体和基础工具函数
- `kernel/light/light.h` - 调用 `area_light_sample`、`area_light_intersect`、`area_light_sample_from_intersection`
- `kernel/light/tree.h` - 调用 `area_light_tree_parameters` 计算光源树重要性
- `kernel/light/background.h` - 门户(portal)采样中使用 `area_light_rect_sample`
- `util/math_intersect.h` - 提供 `ray_quad_intersect` 等几何求交函数
