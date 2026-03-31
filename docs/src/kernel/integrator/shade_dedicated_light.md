# shade_dedicated_light.h - 阴影链接专用光源着色

## 概述

本文件实现了阴影链接（Shadow Linking）功能中专用光源的着色逻辑。当阴影链接交点内核找到需要单独追踪阴影射线的光源后，本内核负责评估该光源的贡献、计算 MIS 权重，并启动专用阴影射线。该文件处理两种类型的光源：分析光源（PRIMITIVE_LAMP）和网格发光体。

## 核心函数

### shadow_linking_light_sample_from_intersection()

- **签名**: `ccl_device_inline bool shadow_linking_light_sample_from_intersection(KernelGlobals kg, const ccl_private Intersection &ccl_restrict isect, const ccl_private Ray &ccl_restrict ray, const float3 N, const uint32_t path_flag, ccl_private LightSample *ccl_restrict ls)`
- **功能**: 从交点数据构建光源样本。区分远距光（调用 `distant_light_sample_from_intersection`）和其他光源（调用 `light_sample_from_intersection`）。

### shadow_linking_light_sample_mis_weight()

- **签名**: `ccl_device_inline float shadow_linking_light_sample_mis_weight(KernelGlobals kg, IntegratorState state, const uint32_t path_flag, const ccl_private LightSample *ls, const float3 P)`
- **功能**: 计算阴影链接光源的 MIS 权重，区分远距光和其他光源类型调用不同的前向 MIS 权重函数。

### shadow_linking_setup_ray_from_intersection()

- **签名**: `ccl_device void shadow_linking_setup_ray_from_intersection(IntegratorState state, ccl_private Ray *ccl_restrict ray, const ccl_private Intersection *ccl_restrict isect)`
- **功能**: 从交点信息设置阴影射线参数。使用保存的主路径自交检查图元（`shadow_link.last_isect_object/prim`）作为射线的自交排除，将交点的图元作为光源自交排除。

### shadow_linking_shade_light()

- **签名**: `ccl_device bool shadow_linking_shade_light(KernelGlobals kg, IntegratorState state, ccl_private Ray &ccl_restrict ray, ccl_private Intersection &ccl_restrict isect, ccl_private ShaderData *emission_sd, ccl_private Spectrum &ccl_restrict bsdf_spectrum, ccl_private float &mis_weight, ccl_private int &ccl_restrict light_group)`
- **功能**: 对分析光源类型的交点执行着色。从交点构建光源样本，评估光源着色器，检查可见性，计算 MIS 权重，最终输出 `bsdf_spectrum = light_eval * mis_weight * dedicated_light_weight`。

### shadow_linking_shade_surface_emission()

- **签名**: `ccl_device bool shadow_linking_shade_surface_emission(KernelGlobals kg, IntegratorState state, ccl_private ShaderData *emission_sd, ccl_global float *ccl_restrict render_buffer, ccl_private Spectrum &ccl_restrict bsdf_spectrum, ccl_private float &mis_weight, ccl_private int &ccl_restrict light_group)`
- **功能**: 对网格发光体执行着色。设置着色器数据，评估表面着色器，检查 `SD_EMISSION` 标志，计算表面自发光的 MIS 权重，输出最终光谱贡献。

### shadow_linking_shade()

- **签名**: `ccl_device void shadow_linking_shade(KernelGlobals kg, IntegratorState state, ccl_global float *ccl_restrict render_buffer)`
- **功能**: 阴影链接着色的主要逻辑。读取交点和射线，根据交点类型（`PRIMITIVE_LAMP` 或网格）选择对应的着色函数。若贡献非零，设置阴影射线并调用 `integrate_direct_light_shadow_init_common` 初始化阴影路径状态。同时处理弹射计数调整、diffuse/glossy 通道权重传递和路径引导记录。

### integrator_shade_dedicated_light()

- **签名**: `ccl_device void integrator_shade_dedicated_light(KernelGlobals kg, IntegratorState state, ccl_global float *ccl_restrict render_buffer)`
- **功能**: 专用光源着色内核的主入口。调用 `shadow_linking_shade` 执行着色，然后调用 `shadow_linking_restore_last_primitives` 恢复主路径的自交检查图元，最后调用 `integrator_shade_surface_next_kernel` 继续主路径。

## 依赖关系

- **内部头文件**:
  - `kernel/light/distant.h` - 远距光采样
  - `kernel/light/light.h` - 光源工具函数
  - `kernel/light/sample.h` - 光源采样和着色器评估
  - `kernel/integrator/shade_surface.h` - 表面着色工具
- **被引用**:
  - `kernel/integrator/megakernel.h` - 超级内核循环
  - `kernel/device/optix/kernel_osl.cu` - OptiX OSL 内核
  - `kernel/device/gpu/kernel.h` - GPU 内核

## 实现细节 / 关键算法

1. **权重乘以命中数**: `dedicated_light_weight` 等于蓄水池采样时的总命中数，用于补偿随机选取一个光源的概率。最终贡献 = 光源评估值 * MIS 权重 * 命中数。
2. **弹射计数调整**: 阴影状态的弹射计数减 1，因为光源累积逻辑（来自 `shade_surface`）会基于实际弹射值做钳制决策。
3. **状态恢复**: 着色完成后必须恢复 `last_isect_object` 和 `last_isect_prim`，因为主路径在返回 `intersect_closest` 状态时需要正确的自交检查图元。
4. **纯体积跳过**: 网格发光体着色时，若着色器标记为 `SD_HAS_ONLY_VOLUME`，跳过表面发射评估。

## 关联文件

- `kernel/integrator/intersect_dedicated_light.h` - 前序的交点计算内核
- `kernel/integrator/shade_surface.h` - 提供 `integrate_direct_light_shadow_init_common` 和后续内核调度
- `kernel/integrator/shadow_linking.h` - 阴影链接工具函数
