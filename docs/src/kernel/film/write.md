# write.h - 渲染缓冲区底层写入与读取基础设施

## 概述

本文件是 Cycles 渲染器胶片模块的基础设施层，提供了渲染缓冲区的像素定位、数据写入和数据读取等底层操作。所有渲染通道的写入（光照通道、数据通道、降噪通道、AOV 通道、自适应采样缓冲区等）都依赖本文件中的函数。

文件区分了 GPU 和 CPU 两种执行环境：GPU 模式下使用原子操作确保多线程安全，CPU 模式下使用普通的累加操作。此外还提供了覆写模式和 RGBE 编解码功能。

## 类与结构体

本文件未定义独立的类或结构体。

## 枚举与常量

- `__ATOMIC_PASS_WRITE__` — 编译宏，在 GPU 内核（`__KERNEL_GPU__`）中自动定义，启用原子写入模式

## 核心函数

### film_pass_pixel_render_buffer() (三个重载)

**重载 1 — 基于积分器状态**:
- **签名**: `ccl_device_forceinline ccl_global float *film_pass_pixel_render_buffer(KernelGlobals kg, ConstIntegratorState state, ccl_global float *ccl_restrict render_buffer)`
- **功能**: 根据积分器路径状态中的 `render_pixel_index` 计算当前像素在渲染缓冲区中的基地址。偏移量 = `render_pixel_index * pass_stride`。

**重载 2 — 基于阴影路径状态**:
- **签名**: `ccl_device_forceinline ccl_global float *film_pass_pixel_render_buffer_shadow(KernelGlobals kg, ConstIntegratorShadowState state, ccl_global float *ccl_restrict render_buffer)`
- **功能**: 与重载 1 类似，但从阴影路径状态读取像素索引。用于直接光照（阴影光线）的缓冲区定位。

**重载 3 — 基于坐标**:
- **签名**: `ccl_device_forceinline ccl_global float *film_pass_pixel_render_buffer(KernelGlobals kg, const int x, const int y, const int offset, const int stride, ccl_global float *ccl_restrict render_buffer)`
- **功能**: 根据像素坐标 (x, y) 和偏移/步幅参数计算缓冲区地址。偏移量 = `(offset + x + y * stride) * pass_stride`。主要用于自适应采样滤波和体积引导降噪等需要访问邻域像素的场景。

### film_write_pass_float()
- **签名**: `ccl_device_inline void film_write_pass_float(ccl_global float *ccl_restrict buffer, const float value)`
- **功能**: 向缓冲区写入（累加）单个浮点值。GPU 模式使用 `atomic_add_and_fetch_float`，CPU 模式使用 `+=`。

### film_write_pass_float3()
- **签名**: `ccl_device_inline void film_write_pass_float3(ccl_global float *ccl_restrict buffer, const float3 value)`
- **功能**: 向缓冲区写入（累加）三分量浮点值。三个分量独立进行原子累加（GPU）或普通累加（CPU）。

### film_write_pass_spectrum()
- **签名**: `ccl_device_inline void film_write_pass_spectrum(ccl_global float *ccl_restrict buffer, Spectrum value)`
- **功能**: 将光谱值转换为 RGB 后写入缓冲区。内部调用 `spectrum_to_rgb()` 和 `film_write_pass_float3()`。

### film_write_pass_float4()
- **签名**: `ccl_device_inline void film_write_pass_float4(ccl_global float *ccl_restrict buffer, const float4 value)`
- **功能**: 向缓冲区写入（累加）四分量浮点值。四个分量独立进行原子或普通累加。

### film_overwrite_pass_rgbe()
- **签名**: `ccl_device_inline void film_overwrite_pass_rgbe(ccl_global float *ccl_restrict buffer, const float3 value)`
- **功能**: 将 RGB 值编码为 RGBE 格式后覆写到缓冲区（单个 float）。不使用原子操作，假设单线程写入。

### film_overwrite_pass_float()
- **签名**: `ccl_device_inline void film_overwrite_pass_float(ccl_global float *ccl_restrict buffer, const float value)`
- **功能**: 直接覆写缓冲区的单个浮点值。用于仅在 sample 0 写入的通道（如深度、物体 ID），不使用原子操作。

### film_overwrite_pass_float3()
- **签名**: `ccl_device_inline void film_overwrite_pass_float3(ccl_global float *ccl_restrict buffer, const float3 value)`
- **功能**: 直接覆写缓冲区的三分量浮点值。

### kernel_read_pass_float()
- **签名**: `ccl_device_inline float kernel_read_pass_float(const ccl_global float *ccl_restrict buffer)`
- **功能**: 从缓冲区读取单个浮点值。

### kernel_read_pass_float3()
- **签名**: `ccl_device_inline float3 kernel_read_pass_float3(const ccl_global float *ccl_restrict buffer)`
- **功能**: 从缓冲区读取三分量浮点值。

### kernel_read_pass_float4()
- **签名**: `ccl_device_inline float4 kernel_read_pass_float4(ccl_global float *ccl_restrict buffer)`
- **功能**: 从缓冲区读取四分量浮点值。

### kernel_read_pass_rgbe()
- **签名**: `ccl_device_inline float3 kernel_read_pass_rgbe(const ccl_global float *ccl_restrict buffer)`
- **功能**: 从缓冲区读取 RGBE 编码的浮点值并解码为 RGB 三通道。

## 依赖关系

- **内部头文件**:
  - `kernel/globals.h` — 全局内核数据（`KernelGlobals`、`kernel_data`）
  - `kernel/integrator/state.h` — 积分器状态类型（`ConstIntegratorState`、`ConstIntegratorShadowState`）
  - `kernel/util/colorspace.h` — 光谱到 RGB 转换（`spectrum_to_rgb()`）
  - `util/types_rgbe.h` — RGBE 类型定义与编解码（`rgb_to_rgbe()`、`rgbe_to_rgb()`）
  - `util/atomic.h` — 原子操作函数（仅 GPU 模式）

- **被引用**:
  - `src/kernel/film/adaptive_sampling.h` — 自适应采样
  - `src/kernel/film/aov_passes.h` — AOV 通道
  - `src/kernel/film/data_passes.h` — 数据通道
  - `src/kernel/film/denoising_passes.h` — 降噪通道
  - `src/kernel/film/light_passes.h` — 光照通道
  - `src/kernel/film/volume_guiding_denoise.h` — 体积引导降噪
  - `src/kernel/integrator/state_flow.h` — 积分器状态流转
  - `src/integrator/path_trace_work_cpu.cpp` — CPU 路径追踪工作器

## 实现细节 / 关键算法

### 原子写入 vs 普通写入

渲染缓冲区的写入模式由执行环境决定：

- **GPU 模式**（`__KERNEL_GPU__` 定义）: 自动定义 `__ATOMIC_PASS_WRITE__`，所有累加写入使用 `atomic_add_and_fetch_float`。这是因为 GPU 上多个线程可能同时写入同一个像素（例如不同的光线路径贡献到同一像素）。
- **CPU 模式**: 使用普通的 `+=` 操作。CPU 渲染通常按像素串行处理，不需要原子操作。

### 覆写 vs 累加

文件提供两类写入语义：
- **累加**（`film_write_pass_*`）: 将新值加到缓冲区现有值上，适用于需要跨采样累积的通道（合成、光照、颜色等）
- **覆写**（`film_overwrite_pass_*`）: 直接替换缓冲区值，不使用原子操作。适用于仅在 sample 0 写入的几何数据通道（深度、物体 ID 等）和 RGBE 编码通道

### 缓冲区布局

渲染缓冲区采用交错布局（interleaved layout）：每个像素的所有通道数据连续存储，`pass_stride` 定义了每个像素的总浮点数。像素 i 的基地址为 `render_buffer + i * pass_stride`，各通道通过偏移量（如 `pass_combined`、`pass_depth` 等）定位。使用 `uint64_t` 进行偏移计算以避免大图像时的整数溢出。

## 关联文件

- `src/kernel/film/adaptive_sampling.h` — 自适应采样收敛判定
- `src/kernel/film/aov_passes.h` — AOV 通道写入
- `src/kernel/film/cryptomatte_passes.h` — Cryptomatte 通道写入
- `src/kernel/film/data_passes.h` — 数据通道写入
- `src/kernel/film/denoising_passes.h` — 降噪通道写入
- `src/kernel/film/light_passes.h` — 光照通道写入
- `src/kernel/film/read.h` — 渲染通道读取（与本文件互为读写对应）
- `src/kernel/film/volume_guiding_denoise.h` — 体积引导降噪
