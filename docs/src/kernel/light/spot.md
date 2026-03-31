# spot.h - 聚光灯采样与求交

## 概述
本文件实现了 Cycles 渲染器中聚光灯(spot light)的采样、求交、衰减和 PDF 计算。聚光灯在点光源的基础上增加了方向性锥体衰减，支持球体光源和传统圆盘光源两种几何模式。聚光灯的锥体衰减使用 `smoothstep` 函数实现柔和的边缘过渡，并支持扩展的采样锥体以在保持物理精度的同时提升采样效率。

## 核心函数

### spot_light_to_local()
- **签名**: `ccl_device float3 spot_light_to_local(KernelGlobals kg, const ccl_global KernelLight *klight, const float3 ray)`
- **功能**: 将世界空间中的向量变换到聚光灯的局部坐标系。使用灯光的逆变换矩阵，并翻转 Z 轴使其指向光照方向。

### spot_light_attenuation()
- **签名**: `ccl_device float spot_light_attenuation(const ccl_global KernelSpotLight *spot, const float3 ray)`
- **功能**: 计算聚光灯的锥体衰减。使用 `smoothstep` 函数在锥角边缘实现平滑过渡，参数 `spot_smooth` 控制过渡宽度。仅依赖局部坐标系的 Z 分量（即与光照轴的夹角余弦）。

### spot_light_uv()
- **签名**: `ccl_device void spot_light_uv(const float3 ray, const float half_cot_half_spot_angle, ccl_private float *u, ccl_private float *v)`
- **功能**: 计算聚光灯的 UV 纹理坐标。使用 `half_cot_half_spot_angle` 缩放因子确保无论聚光角大小，投影的纹理图像始终完整显示。

### spot_light_sample()
- **签名**: `template<bool in_volume_segment> ccl_device_inline bool spot_light_sample(KernelGlobals kg, const ccl_global KernelLight *klight, const float2 rand, const float3 P, const float3 N, const int shader_flags, ccl_private LightSample *ls)`
- **功能**: 在聚光灯上采样一个点。球体模式下：外部时选择可见球体锥与扩展采样锥的较小者进行采样；如果采样扩展锥，则需要与球体求交验证。内部时使用余弦半球或均匀球面采样。传统模式下：在朝向着色点的圆盘上采样。两种模式都计算锥体衰减并在衰减为零时提前返回。

### spot_light_pdf()
- **签名**: `ccl_device_forceinline float spot_light_pdf(const ccl_global KernelSpotLight *spot, const float d_sq, const float r_sq, const float3 N, const float3 D, const uint32_t path_flag)`
- **功能**: 计算聚光灯的采样 PDF。外部时使用可见球体锥与扩展锥的较小者的均匀锥体 PDF，内部时与点光源一致。

### spot_light_mnee_sample_update()
- **签名**: `ccl_device_forceinline void spot_light_mnee_sample_update(KernelGlobals kg, const ccl_global KernelLight *klight, ccl_private LightSample *ls, const float3 P, const float3 N, const uint32_t path_flag)`
- **功能**: 为 MNEE 更新聚光灯采样数据。与点光源的 MNEE 更新逻辑类似，但额外计算锥体衰减和纹理坐标。

### spot_light_intersect()
- **签名**: `ccl_device_inline bool spot_light_intersect(const ccl_global KernelLight *klight, const ccl_private Ray *ccl_restrict ray, ccl_private float *t)`
- **功能**: 光线与聚光灯的求交测试。首先检查光线是否从正面射入（单面光源），然后委托给 `point_light_intersect` 执行实际的几何求交。

### spot_light_sample_from_intersection()
- **签名**: `ccl_device_inline bool spot_light_sample_from_intersection(KernelGlobals kg, const ccl_global KernelLight *klight, const float3 ray_P, const float3 ray_D, const float3 N, const uint32_t path_flag, ccl_private LightSample *ccl_restrict ls)`
- **功能**: 从光线求交结果构建聚光灯采样数据。计算 PDF（球体或圆盘模式）、锥体衰减和纹理坐标。衰减为零时返回 false。

### spot_light_valid_ray_segment()
- **签名**: `ccl_device_inline bool spot_light_valid_ray_segment(KernelGlobals kg, const ccl_global KernelLight *klight, const float3 P, const float3 D, ccl_private Interval<float> *t_range)`
- **功能**: 计算光线被聚光灯照射的有效线段范围。将光线变换到局部空间后与聚光锥体求交，用于体积渲染中的光源贡献范围计算。

### spot_light_tree_parameters()
- **签名**: `template<bool in_volume_segment> ccl_device_forceinline bool spot_light_tree_parameters(const ccl_global KernelLight *klight, const float3 centroid, const float3 P, const ccl_private KernelBoundingCone &bcone, ccl_private float &cos_theta_u, ccl_private float2 &distance, ccl_private float3 &point_to_centroid, ccl_private float &energy)`
- **功能**: 为光源树的重要性计算提供聚光灯参数。几何参数计算与点光源一致，但额外应用 `smoothstep` 衰减来调整能量估计，以更准确地反映聚光灯的方向性。

## 依赖关系
- **内部头文件**: `kernel/light/common.h`, `kernel/light/point.h`, `util/math_fast.h`, `util/math_intersect.h`
- **被引用**: `kernel/light/light.h`, `kernel/light/tree.h`

## 实现细节 / 关键算法
1. **扩展采样锥**: 当球体的可见锥角大于聚光灯的扩展锥角 `cos_half_larger_spread` 时，切换到扩展锥采样。这避免了对大面积可见球体的低效均匀采样，但需要额外的球体求交验证来确保采样方向确实命中光源。
2. **smoothstep 衰减**: `smoothstepf((z - cos_half_spot_angle) * spot_smooth)` 在聚光锥边缘产生平滑衰减。`cos_half_spot_angle` 是锥半角的余弦，`spot_smooth` 是过渡宽度的逆数。
3. **光源树能量估计**: 在光源树参数计算中，通过估算着色点与发射体轴之间的最小角度，并应用 `smoothstep` 衰减来调整能量值。使用 `bcone.theta_e` 代替 `cos_half_spot_angle` 以正确处理非均匀缩放。
4. **单面光源**: `spot_light_intersect` 通过 `dot(ray->D, ray->P - klight->co) >= 0` 判断光线是否从正面射入，确保聚光灯只照射前方。

## 关联文件
- `kernel/light/point.h` - 复用 `point_light_intersect` 进行几何求交
- `kernel/light/common.h` - 提供 `LightSample`、`disk_light_sample`、`light_pdf_area_to_solid_angle`
- `kernel/light/light.h` - 在光源分发中调用聚光灯函数
- `kernel/light/tree.h` - 调用 `spot_light_tree_parameters`
