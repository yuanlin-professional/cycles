# image_sky.h / image_sky.cpp - 天空纹理程序化生成加载器

## 概述

本文件实现了 Nishita 天空模型的纹理加载器 `SkyLoader`，用于程序化生成大气散射天空纹理。它根据太阳高度角、海拔高度和大气密度等参数，预计算天空查找表（LUT），支持单次散射和多次散射两种模式。生成的纹理固定为 512x256 分辨率的 float4 格式。

## 类与结构体

### SkyLoader
- **继承**: `ImageLoader`
- **功能**: 基于 Nishita 天空模型生成大气散射天空纹理 LUT
- **关键成员**:
  - `multiple_scattering` — 是否启用多次散射
  - `sun_elevation` — 太阳高度角
  - `altitude` — 观察者海拔高度
  - `air_density` — 空气密度因子
  - `aerosol_density` — 气溶胶密度因子
  - `ozone_density` — 臭氧密度因子
- **关键方法**:
  - `load_metadata()` — 设置固定元数据：512x256，3 通道，float4 类型
  - `load_pixels()` — 调用 Nishita 天空模型函数预计算天空 LUT 纹理。根据 `multiple_scattering` 标志选择调用 `SKY_multiple_scattering_precompute_texture()` 或 `SKY_single_scattering_precompute_texture()`
  - `name()` — 返回 `"sky_multiple_scattering"`
  - `equals()` — 始终返回 `false`（每个天空纹理参数不同，不可共享）

## 核心函数

无独立核心函数，所有逻辑封装在 `SkyLoader` 类方法中。天空计算的核心逻辑位于外部的 `sky_nishita.h` 头文件中。

## 依赖关系

- **内部头文件**: `scene/image.h`
- **cpp 额外引用**: `sky_nishita.h`（Nishita 天空模型实现）
- **被引用**: `scene/shader_nodes.cpp`（在 SkyTextureNode 中构造 SkyLoader）

## 实现细节 / 关键算法

1. **LUT 分辨率**: 固定 512x256 像素，足够表示天空颜色随方向的变化。
2. **散射模型**: 支持两种模式：
   - 单次散射（`SKY_single_scattering_precompute_texture`）：更快但不够真实
   - 多次散射（`SKY_multiple_scattering_precompute_texture`）：更精确的大气光照
3. **不可共享**: `equals()` 返回 `false`，因为相同的天空参数组合在实际使用中很少重复。每个 Sky 纹理节点都会生成独立的纹理。

## 关联文件

- `scene/image.h/.cpp` — 图像管理系统
- `scene/shader_nodes.cpp` — 着色器节点（SkyTextureNode 使用此加载器）
- `sky_nishita.h` — Nishita 天空散射模型核心算法
