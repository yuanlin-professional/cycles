# read.h - 渲染通道读取与像素数据输出转换

## 概述

本文件实现了从渲染缓冲区读取各类渲染通道数据并转换为最终像素输出的功能。涵盖了标量通道（深度、雾气、采样计数、体积主透射率）、三分量通道（光照通道、法线等 float3 类型）、四分量通道（运动向量、Cryptomatte、合成通道等 float4 类型）的读取与格式化，以及阴影捕捉器的合成计算和自适应采样叠加层的可视化。

文件中的函数负责将原始的累积渲染缓冲区数据根据采样数进行归一化、应用曝光和缩放，转换为可供显示或输出的像素值。

## 类与结构体

本文件未定义新的类或结构体。核心依赖以下外部类型：

- `KernelFilmConvert` — 胶片转换参数结构体，包含各通道偏移、缩放系数、曝光值、组件数量等信息

## 枚举与常量

- `PASS_UNUSED` — 通道未启用哨兵值

## 核心函数

### film_transparency_to_alpha()
- **签名**: `ccl_device_forceinline float film_transparency_to_alpha(const float transparency)`
- **功能**: 将渲染缓冲区中存储的透明度值（transparency = 1 - alpha）转换为 alpha 值，并钳位到 [0, 1] 范围。由于俄罗斯轮盘赌终止策略，alpha 值可能超出正常范围。

### film_get_scale()
- **签名**: `ccl_device_inline float film_get_scale(const ccl_global KernelFilmConvert *ccl_restrict kfilm_convert, const ccl_global float *ccl_restrict buffer)`
- **功能**: 获取当前像素的缩放系数。如果启用了逐像素采样计数（自适应采样），则根据实际采样数调整缩放；否则使用全局缩放值。

### film_get_scale_exposure()
- **签名**: `ccl_device_inline float film_get_scale_exposure(const ccl_global KernelFilmConvert *ccl_restrict kfilm_convert, const ccl_global float *ccl_restrict buffer)`
- **功能**: 获取包含曝光的缩放系数。在 `film_get_scale()` 的基础上可选地乘以曝光值。

### film_get_scale_and_scale_exposure()
- **签名**: `ccl_device_inline bool film_get_scale_and_scale_exposure(const ccl_global KernelFilmConvert *ccl_restrict kfilm_convert, const ccl_global float *ccl_restrict buffer, ccl_private float *ccl_restrict scale, ccl_private float *ccl_restrict scale_exposure)`
- **功能**: 同时获取缩放系数和带曝光的缩放系数。当采样计数为零时返回 `false` 并将两者设为 0，用于短路后续处理。

### film_get_pass_pixel_depth()
- **签名**: `ccl_device_inline void film_get_pass_pixel_depth(const ccl_global KernelFilmConvert *ccl_restrict kfilm_convert, const ccl_global float *ccl_restrict buffer, ccl_private float *ccl_restrict pixel)`
- **功能**: 读取深度通道。深度为 0 时输出极大值 `1e10f`，否则输出缩放后的深度值。

### film_get_pass_pixel_mist()
- **签名**: `ccl_device_inline void film_get_pass_pixel_mist(const ccl_global KernelFilmConvert *ccl_restrict kfilm_convert, const ccl_global float *ccl_restrict buffer, ccl_private float *ccl_restrict pixel)`
- **功能**: 读取雾气通道。由于内核中累积的是 `1 - mist`，此处执行 `saturate(1 - f * scale_exposure)` 还原并钳位。

### film_get_pass_pixel_sample_count()
- **签名**: `ccl_device_inline void film_get_pass_pixel_sample_count(const ccl_global KernelFilmConvert *ccl_restrict kfilm_convert, const ccl_global float *ccl_restrict buffer, ccl_private float *ccl_restrict pixel)`
- **功能**: 读取采样计数通道。将浮点存储的无符号整数通过 `__float_as_uint` 解码后乘以缩放系数。

### film_get_pass_pixel_volume_majorant()
- **签名**: `ccl_device_inline void film_get_pass_pixel_volume_majorant(const ccl_global KernelFilmConvert *ccl_restrict kfilm_convert, const ccl_global float *ccl_restrict buffer, ccl_private float *ccl_restrict pixel)`
- **功能**: 读取体积主透射率通道。使用 `exp(-(f * scale) / count)` 计算透射率。

### film_get_pass_pixel_rgbe()
- **签名**: `ccl_device_inline void film_get_pass_pixel_rgbe(const ccl_global KernelFilmConvert *ccl_restrict kfilm_convert, const ccl_global float *ccl_restrict buffer, ccl_private float *ccl_restrict pixel)`
- **功能**: 读取 RGBE 编码通道。将单个浮点值解码为 RGB 三通道。

### film_get_pass_pixel_float()
- **签名**: `ccl_device_inline void film_get_pass_pixel_float(const ccl_global KernelFilmConvert *ccl_restrict kfilm_convert, const ccl_global float *ccl_restrict buffer, ccl_private float *ccl_restrict pixel)`
- **功能**: 读取通用标量浮点通道。

### film_get_pass_pixel_light_path()
- **签名**: `ccl_device_inline void film_get_pass_pixel_light_path(const ccl_global KernelFilmConvert *ccl_restrict kfilm_convert, const ccl_global float *ccl_restrict buffer, ccl_private float *ccl_restrict pixel)`
- **功能**: 读取光照路径通道（漫反射/光泽/透射的直接/间接分量）。支持：
  - 可选地加上间接光分量（`pass_indirect`）
  - 可选地除以颜色通道（`pass_divide`），用于反照率归一化
  - 可选的 4 通道 alpha 输出

### film_get_pass_pixel_float3()
- **签名**: `ccl_device_inline void film_get_pass_pixel_float3(const ccl_global KernelFilmConvert *ccl_restrict kfilm_convert, const ccl_global float *ccl_restrict buffer, ccl_private float *ccl_restrict pixel)`
- **功能**: 读取通用三分量浮点通道。

### film_get_pass_pixel_motion()
- **签名**: `ccl_device_inline void film_get_pass_pixel_motion(const ccl_global KernelFilmConvert *ccl_restrict kfilm_convert, const ccl_global float *ccl_restrict buffer, ccl_private float *ccl_restrict pixel)`
- **功能**: 读取运动向量通道。使用运动权重通道进行归一化。

### film_get_pass_pixel_cryptomatte()
- **签名**: `ccl_device_inline void film_get_pass_pixel_cryptomatte(const ccl_global KernelFilmConvert *ccl_restrict kfilm_convert, const ccl_global float *ccl_restrict buffer, ccl_private float *ccl_restrict pixel)`
- **功能**: 读取 Cryptomatte 通道。ID 值（x, z 分量）不进行缩放，权重值（y, w 分量）乘以缩放系数。

### film_get_pass_pixel_float4()
- **签名**: `ccl_device_inline void film_get_pass_pixel_float4(const ccl_global KernelFilmConvert *ccl_restrict kfilm_convert, const ccl_global float *ccl_restrict buffer, ccl_private float *ccl_restrict pixel)`
- **功能**: 读取通用四分量浮点通道。RGB 分量使用带曝光缩放，alpha 分量仅使用缩放。

### film_get_pass_pixel_combined()
- **签名**: `ccl_device_inline void film_get_pass_pixel_combined(const ccl_global KernelFilmConvert *ccl_restrict kfilm_convert, const ccl_global float *ccl_restrict buffer, ccl_private float *ccl_restrict pixel)`
- **功能**: 读取合成通道。将透明度转换为 alpha 值。采样计数为零时输出全黑。

### film_calculate_shadow_catcher() / film_calculate_shadow_catcher_denoised()
- **签名**: 见源码
- **功能**: 计算阴影捕捉器通道值。对于非降噪版本：将合成通道减去遮罩贡献后除以阴影捕捉器通道，得到光照比率；再使用透明度进行 alpha-over 混合。降噪版本直接缩放已降噪的阴影捕捉器值。

### film_calculate_shadow_catcher_matte_with_shadow()
- **签名**: `ccl_device_inline float4 film_calculate_shadow_catcher_matte_with_shadow(const ccl_global KernelFilmConvert *ccl_restrict kfilm_convert, const ccl_global float *ccl_restrict buffer)`
- **功能**: 计算带阴影的遮罩合成。使用阴影近似值 `1 - average(shadow_catcher)` 并与遮罩颜色进行 alpha-over 合成。

### film_apply_pass_pixel_overlays_rgba()
- **签名**: `ccl_device_inline void film_apply_pass_pixel_overlays_rgba(const ccl_global KernelFilmConvert *ccl_restrict kfilm_convert, const ccl_global float *ccl_restrict buffer, ccl_private float *ccl_restrict pixel)`
- **功能**: 应用像素叠加层。当启用活跃像素显示时，将仍在采样的像素标记为红色（50% 混合），用于可视化自适应采样的进度。

## 依赖关系

- **内部头文件**:
  - `kernel/types.h` — 内核数据类型定义
  - `util/color.h` — 颜色工具函数

- **被引用**:
  - `src/kernel/device/gpu/kernel.h` — GPU 内核的通道读取入口
  - `src/kernel/device/cpu/kernel_arch_impl.h` — CPU 内核的通道读取入口

## 实现细节 / 关键算法

### 缩放与曝光分离

读取通道时，缩放（scale）和曝光（exposure）被分离处理：
- **缩放**: `1 / sample_count`，用于将累积值归一化为平均值
- **曝光**: 场景曝光设置，仅应用于需要曝光的通道
- 某些通道（如 ID 通道）不应用曝光
- `film_get_scale_and_scale_exposure()` 一次性计算两者，避免重复读取采样计数

### 阴影捕捉器合成

阴影捕捉器的合成算法较为复杂：
1. 从合成通道中减去遮罩对象的贡献，得到不含遮罩的合成结果
2. 将该结果除以阴影捕捉器通道值，得到光照比率（表示阴影程度）
3. 使用合成通道的 alpha 进行 alpha-over 混合：`(1 - alpha) * white + alpha * shadow_catcher`
4. 这使得阴影可以在合成软件中叠加到实拍素材上

## 关联文件

- `src/kernel/film/write.h` — 渲染通道写入（与本文件互为读写对应）
- `src/kernel/film/light_passes.h` — 光照通道写入端
- `src/kernel/film/adaptive_sampling.h` — 自适应采样逻辑
- `src/kernel/device/gpu/kernel.h` — GPU 设备内核
- `src/kernel/device/cpu/kernel_arch_impl.h` — CPU 设备内核
