# volume_guiding_denoise.h - 体积散射概率引导缓冲区降噪滤波

## 概述

本文件实现了体积散射概率引导缓冲区的两趟高斯滤波降噪。体积散射概率引导是一种优化技术，通过预估光线在体积中散射与透射的相对概率，帮助路径追踪积分器更高效地采样体积介质。由于原始的散射/透射缓冲区存在噪声，本文件使用可分离的二维高斯滤波器（先 X 方向后 Y 方向）对其进行平滑处理。

该文件于 2025 年加入，使用 RGBE 编码格式存储降噪后的结果以节省内存。

## 类与结构体

本文件未定义独立的类或结构体。

## 枚举与常量

- 高斯核参数：半径 `radius = 5`，滤波器宽度 `filter_width = 11`（即 2 * 5 + 1）
- 高斯核权重（sigma = 1.5）：预计算的 11 个浮点权重值，总和为 1.0
- `PASS_UNUSED` — 通道未启用哨兵值

相关的渲染缓冲区通道偏移：
- `pass_volume_scatter` — 体积散射原始通道
- `pass_volume_transmit` — 体积透射原始通道
- `pass_volume_scatter_denoised` — 体积散射降噪通道
- `pass_volume_transmit_denoised` — 体积透射降噪通道
- `pass_sample_count` — 采样计数通道

## 核心函数

### volume_guiding_filter_x()
- **签名**: `ccl_device void volume_guiding_filter_x(KernelGlobals kg, ccl_global float *render_buffer, const int y, const int center_x, const int min_x, const int max_x, const int offset, const int stride)`
- **功能**: 对指定像素执行 X 方向的高斯滤波。处理流程：
  1. 以 `center_x` 为中心，在 `[center_x - 5, center_x + 5]` 范围内采样邻域像素
  2. 跳过超出 `[min_x, max_x)` 边界的像素
  3. 读取每个邻域像素的散射和透射原始缓冲区值
  4. 使用高斯权重乘以 `1/sample_count` 进行加权累加
  5. 将结果以 RGBE 编码写入 `pass_volume_scatter_denoised` 和 `pass_volume_transmit_denoised` 通道

### volume_guiding_filter_y()
- **签名**: `ccl_device void volume_guiding_filter_y(KernelGlobals kg, ccl_global float *render_buffer, const int x, const int min_y, const int max_y, const int offset, const int stride)`
- **功能**: 对指定列执行 Y 方向的高斯滤波。处理流程：
  1. 使用循环缓冲区 `scatter_neighbors` / `transmit_neighbors` 存储邻域值，避免在覆写当前行数据时丢失相邻行的原始数据
  2. 从 `pass_volume_scatter_denoised` 和 `pass_volume_transmit_denoised` 通道读取第一趟（X 方向）的输出
  3. 逐行滑动滤波窗口，使用循环索引 `(index + i) % filter_width` 实现高效的窗口滑动
  4. 对卷积结果取绝对值 `fabs()` 后以 RGBE 编码写回

## 依赖关系

- **内部头文件**:
  - `kernel/film/write.h` — 提供 `film_pass_pixel_render_buffer()`（坐标版本）、`film_overwrite_pass_rgbe()`、`kernel_read_pass_float3()`、`kernel_read_pass_rgbe()` 等函数

- **被引用**:
  - `src/kernel/device/gpu/kernel.h` — GPU 内核入口调用滤波函数
  - `src/kernel/device/cpu/kernel_arch_impl.h` — CPU 内核入口调用滤波函数

## 实现细节 / 关键算法

### 可分离高斯滤波

二维高斯滤波被分解为两趟一维滤波（先 X 后 Y），利用高斯核的可分离性将 O(n^2) 的操作降为 O(2n)。高斯核使用 sigma = 1.5 的预计算权重。

### 采样计数归一化

在 X 方向滤波时，每个邻域像素的贡献除以该像素的采样计数（`__float_as_uint(buffer[pass_sample_count])`）。这确保了即使不同像素因自适应采样而有不同的采样数，滤波结果仍然是正确归一化的。

### RGBE 编码

降噪后的散射和透射值使用 RGBE（红绿蓝指数）编码格式存储，每个像素仅占一个 `float`（32 位）。这是一种高动态范围颜色压缩格式，将三通道颜色编码为共享指数的格式，在节省 2/3 内存的同时保留了足够的精度用于体积引导。

### 循环缓冲区优化

Y 方向滤波使用循环缓冲区避免额外的内存分配和数据拷贝。长度为 `filter_width` 的数组通过模运算实现窗口滑动，每次迭代仅需读取一个新的邻域值，效率很高。

### 边界处理

超出图像边界的像素被简单地跳过（X 方向）或填充为零（Y 方向初始化阶段）。这是一种简洁的边界处理策略，在引导概率估计中足够使用。

## 关联文件

- `src/kernel/film/write.h` — 渲染通道读写基础设施，提供 RGBE 编解码函数
- `src/kernel/film/light_passes.h` — `film_write_volume_scattering_guiding_pass()` 负责写入原始的散射/透射通道
- `src/kernel/film/read.h` — `film_get_pass_pixel_volume_majorant()` 读取体积相关通道
- `src/kernel/device/gpu/kernel.h` — GPU 设备内核入口
- `src/kernel/device/cpu/kernel_arch_impl.h` — CPU 设备内核入口
