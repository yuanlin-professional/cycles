# distant.h - 远距光源(平行光)采样与求交

## 概述
本文件实现了 Cycles 渲染器中远距光源(平行光/方向光)的采样、求交和 PDF 计算。远距光源模拟无限远处的光源，光线近似平行，但具有可配置的角直径以模拟太阳等天体光源。采样基于均匀锥体采样，支持光源树的重要性评估。

## 核心函数

### distant_light_uv()
- **签名**: `ccl_device_inline void distant_light_uv(KernelGlobals kg, const ccl_global KernelLight *klight, const float3 D, ccl_private float *u, ccl_private float *v)`
- **功能**: 将光线方向映射到远距光源的 UV 纹理坐标。通过将方向投影到以光源方向为中心的圆盘上计算坐标，用于支持远距光源的纹理映射。坐标采用 Embree/OptiX 兼容的重心坐标表示法。

### distant_light_sample()
- **签名**: `ccl_device_inline bool distant_light_sample(KernelGlobals kg, const ccl_global KernelLight *klight, const float2 rand, ccl_private LightSample *ls)`
- **功能**: 在远距光源的锥体范围内均匀采样一个方向。使用 `sample_uniform_cone` 以光源方向为轴、`one_minus_cosangle` 为锥角参数进行采样。距离设为 FLT_MAX（无限远）。

### distant_light_intersect()
- **签名**: `ccl_device bool distant_light_intersect(const ccl_global KernelLight *klight, const ccl_private Ray *ccl_restrict ray, ccl_private float *t, ccl_private float *u, ccl_private float *v)`
- **功能**: 检测光线是否与远距光源相交（即光线方向是否落在光源的角直径范围内）。角度为零的点光源（无角直径）不可相交。返回 t=FLT_MAX, u=0, v=0 作为阴影光线的优化参数。

### distant_light_sample_from_intersection()
- **签名**: `ccl_device bool distant_light_sample_from_intersection(KernelGlobals kg, const float3 ray_D, const int lamp, ccl_private LightSample *ccl_restrict ls)`
- **功能**: 从前向路径追踪的求交结果构建远距光源的采样数据。仅当光源启用了 MIS 且具有非零角直径时有效。包含 AMD HIP 驱动的特殊兼容处理。

### distant_light_tree_parameters()
- **签名**: `template<bool in_volume_segment> ccl_device_forceinline bool distant_light_tree_parameters(const float3 centroid, const float theta_e, const float t, ccl_private float &cos_theta_u, ccl_private float2 &distance, ccl_private float3 &point_to_centroid, ccl_private float &theta_d)`
- **功能**: 为光源树的重要性计算提供远距光源参数。将远距光视为距离 1 单位的圆盘光，使用 `theta_e` 计算张角余弦。

## 依赖关系
- **内部头文件**: `kernel/geom/object.h`, `kernel/light/common.h`, `util/math_fast.h`
- **被引用**: `kernel/light/light.h`, `kernel/light/tree.h`, `kernel/integrator/shade_dedicated_light.h`

## 实现细节 / 关键算法
1. **均匀锥体采样**: 远距光源在以其方向为轴的锥体内均匀采样。锥角由 `distant.angle` 定义，`one_minus_cosangle` 预计算为 `1 - cos(angle)` 以优化性能。
2. **UV 映射**: 通过逆变换矩阵获取光源局部坐标系的 U 和 V 轴，然后将方向偏差投影到这些轴上。缩放因子 `half_inv_sin_half_angle` 确保 UV 范围覆盖整个圆盘。
3. **体积段处理**: 在世界体积中，远距光可以对程序化生成的体积产生贡献。当射线长度为 FLT_MAX 时使用 1.0 作为默认射线长度，以给远距光适当的权重。
4. **HIP 兼容性**: 包含针对 AMD HIP 驱动 22.10 的 workaround，提前设置 `ls->shader` 以防止某些场景中的挂起问题。

## 关联文件
- `kernel/light/common.h` - 提供 `LightSample` 结构体和 `light_pdf_area_to_solid_angle`
- `kernel/light/light.h` - 在 `light_sample` 和 `lights_intersect_impl` 中分发调用
- `kernel/light/tree.h` - 调用 `distant_light_tree_parameters` 计算光源树重要性
- `kernel/geom/object.h` - 提供 `lamp_get_inverse_transform` 和 `object_lightgroup`
