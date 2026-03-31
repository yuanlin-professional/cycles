# data_passes.h - 几何与材质数据渲染通道写入

## 概述

本文件实现了 Cycles 渲染器中各种几何和材质数据渲染通道的写入逻辑。这些通道包括深度、法线、UV 坐标、运动向量、物体 ID、材质 ID、位置、粗糙度、漫反射/光泽/透射颜色、雾气（Mist）以及 Cryptomatte 通道等。

数据通道主要在光线首次命中表面时写入，用于后期合成和特效制作。文件同时处理了阴影捕捉器（Shadow Catcher）路径的特殊情况，避免数据被重复计数。

## 类与结构体

本文件未定义独立的类或结构体。依赖以下外部类型：

- `ShaderData` — 着色数据，包含命中点的几何信息（位置、法线、UV 等）
- `IntegratorState` — 积分器状态，包含路径标志、吞吐量、采样编号等
- `KernelGlobals` — 全局内核数据

## 枚举与常量

依赖以下路径标志和通道掩码：

- `PATH_RAY_TRANSPARENT_BACKGROUND` — 透明背景光线标志
- `PATH_RAY_SHADOW_CATCHER_PASS` — 阴影捕捉器分裂路径标志
- `PATH_RAY_SINGLE_PASS_DONE` — 单次通道数据已写入标志
- `PASS_ANY` — 任意通道启用标志
- `PASSMASK(...)` — 各种通道的位掩码宏（DEPTH, NORMAL, UV, MOTION, OBJECT_ID, MATERIAL_ID, POSITION, ROUGHNESS 等）
- `CRYPT_OBJECT`, `CRYPT_MATERIAL`, `CRYPT_ASSET` — Cryptomatte 通道类型标志

## 核心函数

### film_write_cryptomatte_pass()
- **签名**: `ccl_device_inline size_t film_write_cryptomatte_pass(ccl_global float *ccl_restrict buffer, const size_t depth, const float id, const float matte_weight)`
- **功能**: 包装函数，调用 `film_write_cryptomatte_slots()` 将 Cryptomatte ID 和权重写入缓冲区，并返回缓冲区步进偏移量 `depth * 4`，使调用者可以依次写入物体、材质和资产三种 Cryptomatte 通道。

### film_write_data_passes()
- **签名**: `ccl_device_inline void film_write_data_passes(KernelGlobals kg, IntegratorState state, const ccl_private ShaderData *sd, ccl_global float *ccl_restrict render_buffer)`
- **功能**: 核心数据通道写入函数。在表面着色时调用，按以下逻辑写入数据：
  1. **前提条件**: 仅在光线穿过透明背景时写入；跳过阴影捕捉器分裂路径
  2. **仅首个采样写入的通道**（sample == 0）: 深度、物体 ID、材质 ID、位置（使用覆写模式 `film_overwrite_pass_*`）
  3. **受 alpha 阈值控制的通道**: 法线、粗糙度、UV、运动向量（在首次命中非透明表面时写入一次，之后设置 `PATH_RAY_SINGLE_PASS_DONE` 标志）
  4. **Cryptomatte 通道**: 根据配置写入物体/材质/资产 Cryptomatte 数据，权重由吞吐量和透明度计算
  5. **颜色通道**: 漫反射颜色、光泽颜色、透射颜色
  6. **雾气通道**: 根据深度、起始距离、衰减参数计算雾气值

### film_write_data_passes_background()
- **签名**: `ccl_device_inline void film_write_data_passes_background(KernelGlobals kg, IntegratorState state, ccl_global float *ccl_restrict render_buffer)`
- **功能**: 当光线到达背景时写入默认数据通道值（深度、物体 ID、材质 ID 写入 0，位置写入零向量）。仅在首个采样时执行，确保背景像素也有有效的数据通道输出。

## 依赖关系

- **内部头文件**:
  - `kernel/integrator/surface_shader.h` — 表面着色器评估函数（`surface_shader_alpha()`、`surface_shader_average_normal()` 等）
  - `kernel/camera/camera.h` — 相机相关函数（`camera_z_depth()`、`camera_distance()`）
  - `kernel/geom/primitive.h` — 几何图元函数（`primitive_uv()`、`primitive_motion_vector()`）
  - `kernel/film/cryptomatte_passes.h` — Cryptomatte 槽位写入
  - `kernel/film/write.h` — 渲染通道写入基础设施

- **被引用**:
  - `src/kernel/integrator/shade_surface.h` — 表面着色阶段调用 `film_write_data_passes()`
  - `src/kernel/integrator/shade_background.h` — 背景着色阶段调用 `film_write_data_passes_background()`

## 实现细节 / 关键算法

### 深度与位置通道的覆写模式

深度、物体 ID、材质 ID 和位置通道仅在 sample == 0 时使用 `film_overwrite_pass_*` 函数写入。这些函数不使用原子累加，而是直接覆写缓冲区值，因为这些数据只需记录首次命中的值。

### 雾气（Mist）计算

雾气计算流程：
1. 将相机距离映射到 `[0, 1]` 范围：`mist = saturate((depth - mist_start) * mist_inv_depth)`
2. 应用衰减曲线：支持线性（1.0）、二次方（2.0）、平方根（0.5）和任意幂次衰减
3. 根据吞吐量和 alpha 值调制：`mist_output = (1 - mist) * average(throughput * alpha)`
4. 最终渲染缓冲区中存储的是 `mist_output` 而非 `1 - mist_output`，取反操作在渲染后进行

### Cryptomatte 权重计算

Cryptomatte 的遮罩权重由 `average(throughput) * (1 - average(transparency))` 计算，即路径吞吐量乘以表面不透明度。这确保了透明表面对 Cryptomatte 遮罩的贡献与其视觉贡献成正比。

## 关联文件

- `src/kernel/film/write.h` — 渲染通道写入基础设施
- `src/kernel/film/cryptomatte_passes.h` — Cryptomatte 底层槽位操作
- `src/kernel/integrator/shade_surface.h` — 表面着色入口
- `src/kernel/integrator/shade_background.h` — 背景着色入口
- `src/kernel/integrator/surface_shader.h` — 表面着色器评估
