# shade_background.h - 背景与远距光着色

## 概述

本文件实现了背景着色内核，处理射线未命中任何几何体时的着色逻辑。它负责评估背景着色器（环境贴图、背景颜色等）、计算远距光（Distant Light）的贡献，以及处理透明背景和 Shadow Catcher 的特殊情况。背景着色是路径追踪中每条射线的潜在终点，需要正确计算 MIS 权重以与直接光照采样保持一致。

## 核心函数

### integrator_eval_background_shader()

- **签名**: `ccl_device Spectrum integrator_eval_background_shader(KernelGlobals kg, IntegratorState state, ccl_global float *ccl_restrict render_buffer)`
- **功能**: 评估背景着色器。执行流程：
  1. 检查光源着色器对当前路径的可见性
  2. 尝试使用快速常量背景色（`surface_shader_constant_emission`）
  3. 若无常量色，设置背景着色数据并执行完整的着色器节点评估
  4. 返回背景着色器的发射光谱

### integrate_background()

- **签名**: `ccl_device_inline void integrate_background(KernelGlobals kg, IntegratorState state, ccl_global float *ccl_restrict render_buffer)`
- **功能**: 处理背景光照的完整逻辑。包括：
  - 透明背景处理：若路径标志含 `PATH_RAY_TRANSPARENT_BACKGROUND`，累积透明度
  - MNEE 防火花：若路径标记了 `PATH_MNEE_CULL_LIGHT_CONNECTION`，跳过使用焦散的背景光
  - AO 弹射因子调整：在 AO 弹射近似模式下乘以 `ao_bounces_factor`
  - MIS 权重计算：调用 `light_sample_mis_weight_forward_background` 计算前向 MIS 权重
  - 路径引导记录：记录背景贡献
  - 写入渲染缓冲区：通过 `film_write_background` 和 `film_write_data_passes_background` 输出

### integrate_distant_lights()

- **签名**: `ccl_device_inline void integrate_distant_lights(KernelGlobals kg, IntegratorState state, ccl_global float *ccl_restrict render_buffer)`
- **功能**: 遍历场景中所有远距光源，检查哪些与当前射线方向匹配。对每个匹配的远距光：
  - 检查可见性标志
  - 检查光链接（Light Linking）匹配
  - 检查阴影链接集合成员资格
  - 检查 MNEE 防火花剔除
  - 评估光源着色器
  - 计算 MIS 权重
  - 记录路径引导数据并写入渲染缓冲区

### integrator_shade_background()

- **签名**: `ccl_device void integrator_shade_background(KernelGlobals kg, IntegratorState state, ccl_global float *ccl_restrict render_buffer)`
- **功能**: 背景着色内核的主入口。依次调用 `integrate_distant_lights` 和 `integrate_background`。处理 Shadow Catcher 背景特殊情况：若路径标志含 `PATH_RAY_SHADOW_CATCHER_BACKGROUND`，在填充背景通道后继续追踪路径（而非终止）。正常情况下调用 `integrator_path_terminate` 终止路径。

## 依赖关系

- **内部头文件**:
  - `kernel/film/data_passes.h` - 数据通道写入
  - `kernel/film/light_passes.h` - 光线通道写入
  - `kernel/integrator/guiding.h` - 路径引导记录
  - `kernel/integrator/intersect_closest.h` - Shadow Catcher 后续内核调度
  - `kernel/integrator/surface_shader.h` - 着色器评估
  - `kernel/light/light.h` - 光源工具
  - `kernel/light/sample.h` - 光源采样和 MIS
  - `kernel/geom/shader_data.h` - 着色数据设置
- **被引用**:
  - `kernel/integrator/megakernel.h` - 超级内核循环
  - `kernel/device/optix/kernel_osl.cu` - OptiX OSL 内核
  - `kernel/device/gpu/kernel.h` - GPU 内核

## 实现细节 / 关键算法

1. **常量背景优化**: `surface_shader_constant_emission` 尝试直接返回背景颜色，避免完整着色器评估的开销。这对纯色背景或简单渐变非常有效。
2. **Shadow Catcher 背景**: Shadow Catcher 分裂路径在渲染背景后需要继续追踪（而不是终止），以计算 Shadow Catcher 对象上的阴影。使用 `PATH_RAY_SHADOW_CATCHER_BACKGROUND` 标志控制此行为。
3. **远距光与背景分离**: 远距光和背景着色器在两个独立的函数中评估（TODO 注释指出未来可能合并为单一循环）。远距光使用 `distant_light_sample_from_intersection` 进行方向匹配。
4. **MNEE 防火花抑制**: 对于标记了 `PATH_MNEE_CULL_LIGHT_CONNECTION` 的路径，若背景光使用 MIS 且存在使用焦散的 `LIGHT_BACKGROUND` 类型光源，则跳过背景评估以避免火花。

## 关联文件

- `kernel/integrator/intersect_closest.h` - 将未命中的射线调度到此内核
- `kernel/integrator/shade_light.h` - 面光源着色（对比参考）
- `kernel/light/sample.h` - MIS 权重计算
