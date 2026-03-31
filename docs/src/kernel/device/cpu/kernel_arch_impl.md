# kernel_arch_impl.h - CPU 内核架构函数实现模板

## 概述

本文件是 CPU 内核所有函数的通用实现模板。与 `kernel_arch.h`（声明模板）配对使用，通过预定义 `KERNEL_ARCH` 宏为不同 CPU 指令集架构生成对应的函数实现。每个 `.cpp` 编译单元（如 `kernel.cpp` 和 `kernel_avx2.cpp`）设置各自的优化标志后包含本文件，实现同一套代码在不同指令集下的编译。支持 `KERNEL_STUB` 模式生成空函数桩。

## 核心函数/宏定义

### 头文件包含体系
在非 `KERNEL_STUB` 模式下包含完整的内核实现：
- `kernel/globals.h` — 内核全局数据
- `kernel/device/cpu/image.h` — CPU 纹理采样
- `kernel/integrator/state*.h` — 积分器状态管理
- `kernel/integrator/init_from_camera.h` / `init_from_bake.h` / `megakernel.h` — 积分器初始化和主循环
- `kernel/film/adaptive_sampling.h` / `cryptomatte_passes.h` / `read.h` / `volume_guiding_denoise.h` — 胶片处理
- `kernel/bake/bake.h` — 烘焙功能

### 积分器函数实现
- **`DEFINE_INTEGRATOR_INIT_KERNEL(name)`**: 定义初始化内核，调用 `integrator_<name>(kg, state, tile, render_buffer, ...)`
- **`DEFINE_INTEGRATOR_SHADE_KERNEL(name)`**: 定义着色内核，调用 `integrator_<name>(kg, state, render_buffer)`
- 实例化: `init_from_camera`, `init_from_bake`, `megakernel`

### 着色器评估函数实现
- **`shader_eval_displace`**: 调用 `kernel_displace_evaluate()`
- **`shader_eval_background`**: 调用 `kernel_background_evaluate()`
- **`shader_eval_curve_shadow_transparency`**: 调用 `kernel_curve_shadow_transparency_evaluate()`
- **`shader_eval_volume_density`**: 调用 `kernel_volume_density_evaluate()`

### 自适应采样函数实现
- **`adaptive_sampling_convergence_check`**: 调用 `film_adaptive_sampling_convergence_check()`
- **`adaptive_sampling_filter_x/y`**: 调用 `film_adaptive_sampling_filter_x/y()`

### Cryptomatte 后处理
- **`cryptomatte_postprocess`**: 调用 `film_cryptomatte_post()`

### 体积引导滤波
- **`volume_guiding_filter_x/y`**: 调用同名的体积引导滤波函数

### 胶片转换函数 (`KERNEL_FILM_CONVERT_FUNCTION(name, is_float)`)
为每种通道类型生成两个函数：
- `film_convert_<name>`: 逐像素循环调用 `film_get_pass_pixel_<name>`
- `film_convert_half_rgba_<name>`: 逐像素转换为半精度 RGBA，对单通道（`is_float=true`）复制到 RGB 三通道，并应用覆盖层

### KERNEL_STUB 模式
当 `KERNEL_STUB` 被定义时，所有函数体替换为 `STUB_ASSERT` 断言，用于不支持特定指令集的平台，确保在误调用时报错。

## 依赖关系

- **内部头文件**:
  - `kernel/device/cpu/compat.h` (兼容层)
  - `kernel/globals.h` (全局数据)
  - `kernel/device/cpu/image.h` (纹理采样)
  - `kernel/integrator/` 系列 (积分器)
  - `kernel/film/` 系列 (胶片处理)
  - `kernel/bake/bake.h` (烘焙)
- **被引用**:
  - `src/kernel/device/cpu/kernel.cpp` (CPU 基准架构)
  - `src/kernel/device/cpu/kernel_avx2.cpp` (AVX2 架构)

## 实现细节 / 关键算法

### Stub 与实际实现的切换
通过 `KERNEL_INVOKE` 宏在 Stub 模式和真实模式间切换：
- Stub 模式: `KERNEL_INVOKE(name, ...) -> STUB_ASSERT(KERNEL_ARCH, name), 0`
- 真实模式: `KERNEL_INVOKE(name, ...) -> integrator_##name(__VA_ARGS__)`

### 胶片转换的逐像素循环
所有胶片转换函数在 CPU 上实现为简单的 for 循环：
```c
for (int i = 0; i < width; i++, buffer += buffer_stride, pixel += pixel_stride) {
    film_get_pass_pixel_<name>(kfilm_convert, buffer, pixel);
}
```
这使得不同指令集优化（SSE4.2 vs AVX2）可以通过编译器自动向量化获得性能提升。

### 宏清理
文件末尾 `#undef` 所有临时宏（`KERNEL_STUB`, `STUB_ASSERT`, `KERNEL_ARCH`, `KERNEL_INVOKE` 等），确保不污染后续编译上下文。

## 关联文件

- `src/kernel/device/cpu/kernel_arch.h` — 对应的函数声明模板
- `src/kernel/device/cpu/kernel.cpp` — 包含本文件生成 cpu 架构实现
- `src/kernel/device/cpu/kernel_avx2.cpp` — 包含本文件生成 cpu_avx2 架构实现
