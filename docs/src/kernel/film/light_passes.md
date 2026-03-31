# light_passes.h - 光照渲染通道累积与写入

## 概述

本文件是 Cycles 渲染器胶片模块中最核心、最庞大的文件，负责光照贡献的评估、钳位（clamping）、各类光照渲染通道的累积以及阴影捕捉器（Shadow Catcher）的处理。它涵盖了从闭包/双向散射分布函数评估结果的漫反射/光泽分量分离，到合成通道（Combined）、自适应采样辅助缓冲区、光照分组（Light Group）、环境光遮蔽（AO）、各类直接/间接光照通道的写入，以及背景、发射、体积散射等多种光照贡献的写入。

文件还包含采样钳位功能，用于限制单次采样贡献的最大值，防止"萤火虫"（firefly）噪声。

## 类与结构体

本文件未定义新的结构体，但大量操作 `BsdfEval` 结构体（假设在其他头文件中定义，包含 `diffuse`、`glossy`、`sum` 三个 `Spectrum` 字段）。

## 枚举与常量

- `PASS_UNUSED` — 通道未启用哨兵值
- `LIGHTGROUP_NONE` — 不属于任何光照分组
- `PATH_RAY_SHADOW_FOR_AO` — 用于环境光遮蔽的阴影光线
- `PATH_RAY_ANY_PASS` — 光线经过至少一次弹射
- `PATH_RAY_SURFACE_PASS` / `PATH_RAY_VOLUME_PASS` — 表面/体积弹射通道标志
- `PATH_RAY_SHADOW_CATCHER_HIT` / `PATH_RAY_SHADOW_CATCHER_BACKGROUND` — 阴影捕捉器相关标志
- `PATH_RAY_VOLUME_PRIMARY_TRANSMIT` / `PATH_RAY_VOLUME_SCATTER` — 体积主透射/散射标志
- `PASSMASK(COMBINED)` — 合成通道位掩码

## 核心函数

### bsdf_eval_init() (两个重载)
- **签名**: `ccl_device_inline void bsdf_eval_init(ccl_private BsdfEval *eval, const ccl_private ShaderClosure *sc, const float3 wo, Spectrum value)` 及 `bsdf_eval_init(ccl_private BsdfEval *eval, Spectrum value)`
- **功能**: 初始化闭包/双向散射分布函数评估结果。根据闭包类型将值分配到漫反射或光泽分量。玻璃类闭包根据出射方向与法线的点积判定归属。不带闭包参数的版本仅设置总和。

### bsdf_eval_accum() (两个重载)
- **签名**: `ccl_device_inline void bsdf_eval_accum(ccl_private BsdfEval *eval, const ccl_private ShaderClosure *sc, const float3 wo, Spectrum value)` 及 `bsdf_eval_accum(ccl_private BsdfEval *eval, Spectrum value)`
- **功能**: 在已有评估结果上累加新的闭包贡献。分类逻辑与初始化相同。

### bsdf_eval_is_zero()
- **签名**: `ccl_device_inline bool bsdf_eval_is_zero(ccl_private BsdfEval *eval)`
- **功能**: 判断评估结果是否为零。

### bsdf_eval_mul() (两个重载)
- **签名**: `ccl_device_inline void bsdf_eval_mul(ccl_private BsdfEval *eval, const float value)` 及 `bsdf_eval_mul(ccl_private BsdfEval *eval, Spectrum value)`
- **功能**: 将评估结果的所有分量乘以标量或光谱值。

### bsdf_eval_sum()
- **签名**: `ccl_device_inline Spectrum bsdf_eval_sum(const ccl_private BsdfEval *eval)`
- **功能**: 返回评估结果的总和。

### bsdf_eval_pass_diffuse_weight() / bsdf_eval_pass_glossy_weight()
- **签名**: `ccl_device_inline Spectrum bsdf_eval_pass_diffuse_weight(const ccl_private BsdfEval *eval)` / `ccl_device_inline Spectrum bsdf_eval_pass_glossy_weight(const ccl_private BsdfEval *eval)`
- **功能**: 计算漫反射/光泽分量在总和中的比例权重。用于后续将光照贡献分配到对应的渲染通道。透射权重通过 `1 - diffuse_weight - glossy_weight` 隐式计算。

### film_clamp_light()
- **签名**: `ccl_device_forceinline void film_clamp_light(KernelGlobals kg, ccl_private Spectrum *L, const int bounce)`
- **功能**: 对光照贡献进行钳位处理。首先确保值有限（`ensure_finite`），然后根据弹射次数使用直接光钳位阈值或间接光钳位阈值限制总亮度。

### film_write_sample()
- **签名**: `ccl_device_inline int film_write_sample(KernelGlobals kg, ConstIntegratorState state, ccl_global float *ccl_restrict render_buffer, const int sample, const int sample_offset)`
- **功能**: 原子递增像素的采样计数，返回当前采样编号。用于自适应采样中的逐像素采样计数追踪。

### film_write_adaptive_buffer()
- **签名**: `ccl_device void film_write_adaptive_buffer(KernelGlobals kg, const int sample, const Spectrum contribution, ccl_global float *ccl_restrict buffer)`
- **功能**: 为自适应采样写入辅助缓冲区。仅对 Class A 采样写入（值乘以 2.0），实现半数采样估计，供收敛判定使用。

### film_write_volume_scattering_guiding_pass()
- **签名**: `ccl_device_inline void film_write_volume_scattering_guiding_pass(KernelGlobals kg, ccl_global float *ccl_restrict buffer, const uint32_t path_flag, const Spectrum contribution)`
- **功能**: 写入体积散射概率引导通道。根据路径标志分别写入体积透射或体积散射通道。

### film_write_shadow_catcher() / film_write_shadow_catcher_transparent() / film_write_shadow_catcher_transparent_only()
- **签名**: 多个签名，见源码
- **功能**: 处理阴影捕捉器的光照贡献写入。遮罩路径（matte path）的贡献写入 `pass_shadow_catcher_matte` 通道；阴影捕捉器物体路径写入 `pass_shadow_catcher` 通道并返回 `true` 以阻止写入合成通道。

### film_write_shadow_catcher_bounce_data()
- **签名**: `ccl_device_forceinline void film_write_shadow_catcher_bounce_data(KernelGlobals kg, IntegratorState state, ccl_global float *ccl_restrict render_buffer)`
- **功能**: 在阴影捕捉器物体上弹射时写入采样计数和透明度数据。

### film_write_combined_pass()
- **签名**: `ccl_device_inline void film_write_combined_pass(KernelGlobals kg, const uint32_t path_flag, const int sample, const Spectrum contribution, ccl_global float *ccl_restrict buffer)`
- **功能**: 写入合成通道。首先尝试阴影捕捉器处理，然后写入合成通道、自适应采样缓冲区和体积散射引导通道。

### film_write_combined_transparent_pass()
- **签名**: `ccl_device_inline void film_write_combined_transparent_pass(KernelGlobals kg, const uint32_t path_flag, const int sample, const Spectrum contribution, const float transparent, ccl_global float *ccl_restrict buffer)`
- **功能**: 带透明度的合成通道写入版本。

### film_write_emission_or_background_pass()
- **签名**: `ccl_device_inline void film_write_emission_or_background_pass(KernelGlobals kg, ConstIntegratorState state, Spectrum contribution, ccl_global float *ccl_restrict buffer, const int pass, const int lightgroup)`
- **功能**: 将发射或背景光照贡献写入对应的渲染通道。对于间接可见的贡献，根据路径类型（表面反射/体积）和弹射次数分配到漫反射直接/间接、光泽直接/间接、透射直接/间接、体积直接/间接通道。同时处理降噪反照率和光照分组。

### film_write_direct_light()
- **签名**: `ccl_device_inline void film_write_direct_light(KernelGlobals kg, ConstIntegratorShadowState state, ccl_global float *ccl_restrict render_buffer)`
- **功能**: 写入直接光照（阴影光线计算结果）。处理环境光遮蔽和各类光照通道的分配，逻辑与 `film_write_emission_or_background_pass()` 类似但从阴影路径状态读取数据。

### film_write_transparent()
- **签名**: `ccl_device_inline void film_write_transparent(KernelGlobals kg, const uint32_t path_flag, const float transparent, ccl_global float *ccl_restrict buffer)`
- **功能**: 写入透明度到合成通道的 alpha 分量。注意缓冲区中累积的是透明度（1 - alpha）而非 alpha 值。

### film_write_holdout()
- **签名**: `ccl_device_inline void film_write_holdout(KernelGlobals kg, ConstIntegratorState state, const uint32_t path_flag, const float transparent, ccl_global float *ccl_restrict render_buffer)`
- **功能**: 写入 Holdout（完全透明区域）的透明度。

### film_write_background()
- **签名**: `ccl_device_inline void film_write_background(KernelGlobals kg, ConstIntegratorState state, const Spectrum L, const float transparent, const bool is_transparent_background_ray, ccl_global float *ccl_restrict render_buffer)`
- **功能**: 写入背景光照贡献。对透明背景光线仅写入透明度，否则写入合成通道和背景通道。

### film_write_volume_emission() / film_write_surface_emission()
- **签名**: 见源码
- **功能**: 分别写入体积发射和表面发射光照贡献到合成通道及发射通道。

## 依赖关系

- **内部头文件**:
  - `kernel/film/write.h` — 渲染通道写入基础设施
  - `kernel/integrator/shadow_catcher.h` — 阴影捕捉器辅助函数
  - `kernel/sample/pattern.h` — 采样分类函数（`sample_is_class_A()`）
  - `util/atomic.h` — 原子操作

- **被引用**:
  - `src/kernel/integrator/surface_shader.h` — 调用 `bsdf_eval_*` 系列函数
  - `src/kernel/integrator/volume_shader.h` — 调用 `bsdf_eval_*` 系列函数
  - `src/kernel/integrator/shade_surface.h` — 调用光照通道写入函数
  - `src/kernel/integrator/shade_volume.h` — 调用体积发射和直接光写入
  - `src/kernel/integrator/shade_background.h` — 调用背景和 holdout 写入
  - `src/kernel/integrator/shade_light.h` — 调用发射写入
  - `src/kernel/integrator/init_from_bake.h` — 烘焙初始化
  - `src/kernel/integrator/init_from_camera.h` — 相机路径初始化
  - `src/kernel/integrator/intersect_closest.h` — 最近交点处理

## 实现细节 / 关键算法

### 闭包/双向散射分布函数分量分离

光照渲染通道需要将总光照贡献拆分为漫反射和光泽两个分量。`BsdfEval` 在初始化和累加时，根据闭包类型自动分类：
- 漫反射类闭包 -> `eval->diffuse`
- 光泽类闭包 -> `eval->glossy`
- 玻璃类闭包 -> 根据出射方向判定（反射面归光泽，折射面不归类）
- 透射权重不显式存储，而是在写入通道时通过 `1 - diffuse - glossy` 隐式计算，以节省 GPU 显存

### 采样钳位

钳位在每次光照贡献写入前执行（`film_clamp_light`）：
1. 首先用 `ensure_finite()` 确保值有限
2. 根据弹射次数选择直接光或间接光钳位阈值
3. 计算光谱分量绝对值之和，若超过阈值则等比缩放

### 光照通道路由

光照贡献的通道路由基于路径标志：
- 直接可见（无弹射）：写入发射或背景通道
- 经表面反射（`PATH_RAY_SURFACE_PASS`）：按漫反射/光泽/透射权重分配到对应的直接或间接通道
- 经体积散射（`PATH_RAY_VOLUME_PASS`）：写入体积直接或间接通道
- 弹射次数为 0/1 决定写入直接通道还是间接通道

## 关联文件

- `src/kernel/film/write.h` — 底层渲染缓冲区写入
- `src/kernel/film/adaptive_sampling.h` — 自适应采样收敛判定
- `src/kernel/film/denoising_passes.h` — 降噪特征通道
- `src/kernel/film/data_passes.h` — 数据通道
- `src/kernel/integrator/shadow_catcher.h` — 阴影捕捉器逻辑
- `src/kernel/sample/pattern.h` — 采样模式
