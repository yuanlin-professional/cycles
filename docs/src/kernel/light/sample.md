# sample.h - 光源采样高层接口与MIS权重计算

## 概述
本文件是 Cycles 渲染器光源采样的最高层接口，整合了光源选择（光源树或分布采样）、着色器求值、阴影光线生成以及多重重要性采样(MIS)权重计算。它是积分器(integrator)与底层光源实现之间的桥梁，提供从着色点或体积段采样光源的完整工作流。

## 核心函数

### light_sample_shader_eval()
- **签名**: `ccl_device_noinline_cpu Spectrum light_sample_shader_eval(KernelGlobals kg, IntegratorState state, ccl_private ShaderData *ccl_restrict emission_sd, ccl_private LightSample *ccl_restrict ls, const float time)`
- **功能**: 在采样的光源位置上求值着色器。对于常量发射着色器直接返回结果，否则设置着色数据并调用 `surface_shader_eval` 进行完整的着色器求值。背景光使用 `shader_setup_from_background`，其他光源使用 `shader_setup_from_sample`。最终结果乘以 `eval_fac` 和光源强度。

### light_sample_terminate()
- **签名**: `ccl_device_inline bool light_sample_terminate(KernelGlobals kg, ccl_private BsdfEval *ccl_restrict eval, const float rand_terminate)`
- **功能**: 阴影光线的早期路径终止判断。当 BSDF 求值结果为零时直接终止；启用俄罗斯轮盘赌时，低贡献路径有概率被终止，幸存路径的权重相应提升。

### shadow_ray_smooth_surface_offset()
- **签名**: `ccl_device_inline float3 shadow_ray_smooth_surface_offset(KernelGlobals kg, const ccl_private ShaderData *ccl_restrict sd, const float3 Ng)`
- **功能**: 计算平滑表面的光线偏移量以避免阴影终止器(shadow terminator)瑕疵。使用抛物面近似对三角形表面进行局部提升，结合线性包络确保偏移量的上下界。

### shadow_ray_offset()
- **签名**: `ccl_device_inline float3 shadow_ray_offset(KernelGlobals kg, const ccl_private ShaderData *ccl_restrict sd, const float3 L, ccl_private bool *r_skip_self)`
- **功能**: 计算阴影光线的起点偏移以避免阴影终止器瑕疵。仅对光滑法线三角形且偏移阈值大于 0 时生效。偏移量与入射角和几何法线的关系决定，在阈值附近平滑过渡。

### shadow_ray_setup()
- **签名**: `ccl_device_inline void shadow_ray_setup(const ccl_private ShaderData *ccl_restrict sd, const ccl_private LightSample *ccl_restrict ls, const float3 P, ccl_private Ray *ray, const bool skip_self)`
- **功能**: 设置阴影光线的几何参数。远距光使用光源方向和无限距离，其他光源使用归一化的方向和有限距离。当着色器不投射阴影时设置零光线作为信号。

### light_sample_to_surface_shadow_ray()
- **签名**: `ccl_device_inline void light_sample_to_surface_shadow_ray(KernelGlobals kg, const ccl_private ShaderData *ccl_restrict sd, const ccl_private LightSample *ccl_restrict ls, ccl_private Ray *ray)`
- **功能**: 从表面着色点创建指向光源采样点的阴影光线，包含自动偏移。

### light_sample_to_volume_shadow_ray()
- **签名**: `ccl_device_inline void light_sample_to_volume_shadow_ray(const ccl_private ShaderData *ccl_restrict sd, const ccl_private LightSample *ccl_restrict ls, const float3 P, ccl_private Ray *ray)`
- **功能**: 从体积散射点创建指向光源采样点的阴影光线，不需要偏移处理。

### light_sample_mis_weight_forward()
- **签名**: `ccl_device_inline float light_sample_mis_weight_forward(KernelGlobals kg, const float forward_pdf, const float nee_pdf)`
- **功能**: 计算前向路径追踪的 MIS 权重。使用幂启发式(power heuristic)平衡前向 PDF 和 NEE PDF。调试模式下支持强制使用单一策略。

### light_sample_mis_weight_nee()
- **签名**: `ccl_device_inline float light_sample_mis_weight_nee(KernelGlobals kg, const float nee_pdf, const float forward_pdf)`
- **功能**: 计算下一事件估计(NEE)的 MIS 权重。是 `light_sample_mis_weight_forward` 的对偶。

### light_sample_from_volume_segment()
- **签名**: `ccl_device_inline bool light_sample_from_volume_segment(KernelGlobals kg, const float3 rand, const float time, const float3 P, const float3 D, const float t, const int object_receiver, const int bounce, const uint32_t path_flag, ccl_private LightSample *ls)`
- **功能**: 从体积段采样光源。使用光源树或分布采样选择光源，然后在选中的光源上采样位置。

### light_sample_from_position()
- **签名**: `ccl_device bool light_sample_from_position(KernelGlobals kg, const float3 rand, const float time, const float3 P, const float3 N, const int object_receiver, const int shader_flags, const int bounce, const uint32_t path_flag, ccl_private LightSample *ls)`
- **功能**: 从表面位置采样光源。与体积段版本类似但传递法线而非方向。

### light_sample_update()
- **签名**: `ccl_device_forceinline void light_sample_update(KernelGlobals kg, ccl_private LightSample *ls, const float3 P, const float3 N, const uint32_t path_flag)`
- **功能**: 为 MNEE 更新光源采样数据。根据光源类型分发到对应的更新函数，重新应用选择 PDF。

### light_sample_mis_weight_forward_surface()
- **签名**: `ccl_device_inline float light_sample_mis_weight_forward_surface(KernelGlobals kg, IntegratorState state, const uint32_t path_flag, const ccl_private ShaderData *sd)`
- **功能**: 计算表面前向采样的 MIS 权重。结合三角形光源 PDF 和光源选择 PDF（光源树或分布），与路径的 BSDF PDF 进行 MIS 平衡。

### light_sample_mis_weight_forward_lamp()
- **签名**: `ccl_device_inline float light_sample_mis_weight_forward_lamp(KernelGlobals kg, IntegratorState state, const uint32_t path_flag, const ccl_private LightSample *ls, const float3 P)`
- **功能**: 计算灯光前向采样的 MIS 权重。

### light_sample_mis_weight_forward_background()
- **签名**: `ccl_device_inline float light_sample_mis_weight_forward_background(KernelGlobals kg, IntegratorState state, const uint32_t path_flag)`
- **功能**: 计算背景光前向采样的 MIS 权重。

## 依赖关系
- **内部头文件**: `kernel/integrator/surface_shader.h`, `kernel/light/distribution.h`, `kernel/light/light.h`, `kernel/types.h`, `kernel/light/tree.h`(条件编译), `kernel/geom/shader_data.h`, `kernel/sample/mis.h`
- **被引用**: `kernel/integrator/shade_surface.h`, `kernel/integrator/shade_volume.h`, `kernel/integrator/mnee.h`, `kernel/integrator/shade_background.h`, `kernel/integrator/shade_dedicated_light.h`, `kernel/integrator/shade_light.h`

## 实现细节 / 关键算法
1. **阴影终止器偏移**: 使用三角形顶点法线与位置的抛物面近似，结合线性包络计算安全的光线起点偏移。`offset_cutoff` 控制偏移影响的范围（如 0.1 表示 10-20% 的光线受影响），在阈值附近使用 `clamp` 实现平滑过渡。
2. **MIS 权重**: 使用幂启发式 `power_heuristic(pdf_a, pdf_b) = pdf_a^2 / (pdf_a^2 + pdf_b^2)` 平衡前向和 NEE 策略。调试模式支持强制使用单一策略进行对比分析。
3. **着色器求值优化**: 对常量发射着色器（`surface_shader_constant_emission`）跳过完整的着色器求值管线，直接返回常量值，提升 GPU 的执行一致性和编译时间。
4. **Sobol 序列优化**: 在光源采样中优先使用 Sobol 序列的前两个维度（具有更好的分层特性）用于光源表面上的位置采样，第三维度用于光源选择。

## 关联文件
- `kernel/light/light.h` - 提供 `light_sample` 和 `light_sample_from_intersection`
- `kernel/light/distribution.h` - 提供分布采样的 `light_distribution_sample`
- `kernel/light/tree.h` - 提供光源树的 `light_tree_sample` 和 `light_tree_pdf`
- `kernel/sample/mis.h` - 提供 `power_heuristic` 函数
- `kernel/integrator/surface_shader.h` - 提供着色器求值接口
