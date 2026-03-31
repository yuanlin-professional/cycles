# guiding.h - 路径引导系统

## 概述

本文件实现了 Cycles 的路径引导（Path Guiding）功能，基于 Intel Open Path Guiding Library (OpenPGL) 构建。路径引导通过学习场景中的辐射分布来优化采样方向的选择，从而加速收敛。该文件包含路径段记录、引导场查询、RIS（重要性重采样）目标计算，以及用于 BSDF 和体积相函数的引导采样功能。所有功能通过 `__PATH_GUIDING__` 宏和 `PATH_GUIDING_LEVEL` 分级控制。

## 核心函数

### calculate_ris_target()

- **签名**: `ccl_device_forceinline bool calculate_ris_target(ccl_private GuidingRISSample *ris_sample, const ccl_private float guiding_sampling_prob)`
- **功能**: 计算 RIS（Resampled Importance Sampling）的目标函数值、RIS 采样 PDF 和 RIS 权重。结合 BSDF 求值和入射辐射 PDF，用于在 BSDF 采样和引导采样之间进行重要性重采样。

### guiding_record_surface_segment()

- **签名**: `ccl_device_forceinline void guiding_record_surface_segment(KernelGlobals kg, IntegratorState state, const ccl_private ShaderData *sd)`
- **功能**: 在当前表面交点处创建并记录一个新的路径段。初始化路径段的位置、出射方向、散射贡献等字段。该函数通常在 `guiding_record_surface_bounce` 之前调用。

### guiding_record_surface_bounce()

- **签名**: `ccl_device_forceinline void guiding_record_surface_bounce(KernelGlobals kg, IntegratorState state, const Spectrum weight, const float pdf, const float3 N, const float3 wo, const float2 roughness, const float eta)`
- **功能**: 记录当前路径段处的表面散射事件，包括散射权重、PDF、法线、出射方向、粗糙度和 IOR（折射率）。

### guiding_record_surface_emission()

- **签名**: `ccl_device_forceinline void guiding_record_surface_emission(KernelGlobals kg, IntegratorState state, const Spectrum Le, const float mis_weight)`
- **功能**: 记录当前表面交点处的自发光贡献和 MIS 权重。

### guiding_record_bssrdf_segment() / guiding_record_bssrdf_weight() / guiding_record_bssrdf_bounce()

- **功能**: 记录次表面散射（BSSRDF）的路径段、传输权重和散射方向。BSSRDF 入口点被标记为体积散射类型。

### guiding_record_volume_segment() / guiding_record_volume_bounce() / guiding_record_volume_transmission() / guiding_record_volume_emission()

- **功能**: 记录体积交互相关的路径段，包括体积散射事件、透射权重和体积自发光。

### guiding_record_light_surface_segment()

- **签名**: `ccl_device_forceinline void guiding_record_light_surface_segment(KernelGlobals kg, IntegratorState state, const ccl_private Intersection *ccl_restrict isect)`
- **功能**: 在光源交点处添加伪路径段（如面光源、球光源），用于路径引导的光源贡献训练。

### guiding_record_background()

- **签名**: `ccl_device_forceinline void guiding_record_background(KernelGlobals kg, IntegratorState state, const Spectrum L, const float mis_weight)`
- **功能**: 当路径离开场景并击中背景光（环境贴图、远距光等）时，记录最终路径段。

### guiding_record_direct_light()

- **签名**: `ccl_device_forceinline void guiding_record_direct_light(KernelGlobals kg, IntegratorShadowState state)`
- **功能**: 记录通过下一事件估计（NEE）或专用 BSDF 阴影射线获得的直接光照贡献。

### guiding_record_continuation_probability()

- **签名**: `ccl_device_forceinline void guiding_record_continuation_probability(KernelGlobals kg, IntegratorState state, const float continuation_probability)`
- **功能**: 记录俄罗斯轮盘赌的路径继续概率。

### guiding_write_debug_passes()

- **签名**: `ccl_device_forceinline void guiding_write_debug_passes(KernelGlobals kg, IntegratorState state, const ccl_private ShaderData *sd, ccl_global float *ccl_restrict render_buffer)`
- **功能**: 将路径引导相关的调试信息（引导概率、平均粗糙度等）写入独立的渲染通道。仅在首次弹射且启用 `WITH_CYCLES_DEBUG` 时有效。

### guiding_bsdf_init() / guiding_bsdf_sample() / guiding_bsdf_pdf()

- **功能**: 初始化引导 BSDF 采样分布，应用余弦乘积；采样引导方向并返回对应 PDF；查询给定方向的引导 PDF。

### guiding_surface_incoming_radiance_pdf()

- **签名**: `ccl_device_forceinline float guiding_surface_incoming_radiance_pdf(KernelGlobals kg, const float3 wo)`
- **功能**: 查询给定方向的入射辐射 PDF，用于 RIS 目标函数计算。

### guiding_phase_init() / guiding_phase_sample() / guiding_phase_pdf()

- **功能**: 初始化引导体积相函数分布（应用 Henyey-Greenstein 乘积），采样引导方向和查询 PDF。对于接近 delta 的相函数（|g| >= 0.99）不启用引导。

## 依赖关系

- **内部头文件**:
  - `kernel/globals.h` - 内核全局数据
  - `kernel/types.h` - 内核类型定义
  - `kernel/integrator/state.h` - 积分器状态
  - `kernel/util/colorspace.h` - 色彩空间转换
  - `kernel/closure/bsdf.h` - BSDF 闭包
  - `util/color.h` - 颜色工具
- **被引用**:
  - `kernel/integrator/intersect_closest.h` - 最近交点计算
  - `kernel/integrator/shade_background.h` - 背景着色
  - `kernel/integrator/shade_shadow.h` - 阴影着色
  - `kernel/integrator/shade_surface.h` - 表面着色
  - `kernel/integrator/shade_volume.h` - 体积着色
  - `kernel/integrator/surface_shader.h` - 表面着色器
  - `kernel/integrator/volume_shader.h` - 体积着色器
  - `kernel/integrator/subsurface_disk.h` / `subsurface_random_walk.h` - 次表面散射
  - `scene/integrator.h` / `integrator/path_trace.h` - 场景层积分器

## 实现细节 / 关键算法

1. **GuidingRISSample 结构**: 封装了一次 RIS 采样所需的全部信息，包括随机数、粗糙度、IOR、标签、BSDF/引导 PDF、RIS 目标/PDF/权重、入射辐射 PDF 和 BSDF 求值结果。
2. **RIS 目标函数**: `ris_target = avg_bsdf_eval * ((1 - p) * 1/(2*pi) + p * incoming_radiance_pdf)`，其中 `p` 为引导采样概率。RIS PDF 为 BSDF PDF 和引导 PDF 的均匀混合：`ris_pdf = 0.5 * (bsdf_pdf + guide_pdf)`。
3. **分级控制**: `PATH_GUIDING_LEVEL` 控制功能粒度。Level 1 提供基本路径记录，Level 4 启用完整的 BSDF/体积引导采样和调试通道。
4. **OpenPGL 集成**: 所有路径段操作通过 `openpgl::cpp` 命名空间的 API 完成，如 `SetPosition`、`SetDirectionIn`、`SetScatteringWeight` 等。
5. **Shadow Catcher 排除**: 所有记录函数在路径标志包含 `PATH_RAY_SHADOW_CATCHER_PASS` 时跳过，以避免污染引导训练数据。

## 关联文件

- `kernel/integrator/shade_surface.h` - 在表面着色中调用引导记录和采样
- `kernel/integrator/shade_volume.h` - 在体积着色中调用引导记录和采样
- `kernel/integrator/intersect_closest.h` - 在路径终止判断中记录继续概率
