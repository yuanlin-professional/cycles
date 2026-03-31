# denoising_passes.h - 降噪特征通道写入

## 概述

本文件实现了 Cycles 渲染器中降噪器所需的特征通道（feature passes）的写入逻辑。降噪器利用这些辅助信息——包括降噪深度、降噪法线和降噪反照率（albedo）——来区分噪声和真实的场景细节，从而更有效地去除蒙特卡洛渲染噪声。

文件分别处理了表面命中和体积散射两种场景下的降噪特征写入。整段代码由 `__DENOISING_FEATURES__` 编译宏保护。

## 类与结构体

本文件未定义独立的类或结构体。依赖以下外部类型：

- `ShaderData` — 着色数据，提供命中点几何信息和闭包/双向散射分布函数列表
- `ShaderClosure` — 单个闭包/双向散射分布函数，包含法线、采样权重和类型等信息
- `IntegratorState` — 积分器状态
- `BsdfEval` — 双向散射分布函数评估结果（在 light_passes.h 中定义）

## 枚举与常量

- `PATH_RAY_DENOISING_FEATURES` — 路径标志，表示当前路径仍需写入降噪特征
- `PATH_RAY_SHADOW_CATCHER_PASS` — 阴影捕捉器分裂路径标志
- `SD_HAS_ONLY_VOLUME` — 着色数据标志，表示该命中点仅包含体积数据
- `CLOSURE_BSDF_HAIR_HUANG_ID` — 黄氏毛发闭包/双向散射分布函数类型 ID
- `PASS_UNUSED` — 通道未启用哨兵值

## 核心函数

### film_write_denoising_features_surface()
- **签名**: `ccl_device_forceinline void film_write_denoising_features_surface(KernelGlobals kg, IntegratorState state, const ccl_private ShaderData *sd, ccl_global float *ccl_restrict render_buffer)`
- **功能**: 在表面命中时写入降噪特征通道。核心逻辑：
  1. 跳过不需要降噪特征的路径（标志位检查）
  2. 跳过仅含体积的隐式透明表面和阴影捕捉器路径
  3. **降噪深度**: 使用 `denoising_feature_throughput * (ray_length - tmin)` 计算加权深度
  4. **法线与反照率**: 遍历所有闭包/双向散射分布函数，累加法线（按采样权重加权）并区分漫反射和镜面反射反照率
  5. **镜面反射延迟策略**: 当 75% 或更多的采样权重属于镜面反射类闭包时，不写入特征而是将镜面反射反照率乘入 `denoising_feature_throughput`，延迟到下一次弹射再写入
  6. 法线变换到相机空间后写入

### film_write_denoising_features_volume()
- **签名**: `ccl_device_forceinline void film_write_denoising_features_volume(KernelGlobals kg, IntegratorState state, const Spectrum albedo, const bool scatter, ccl_global float *ccl_restrict render_buffer)`
- **功能**: 在体积散射时写入降噪特征通道。体积散射被假定为充分漫反射的，因此：
  1. 法线写入为视线方向 `(0, 0, -1)`（相机空间），并清除降噪特征标志
  2. 反照率写入为 `denoising_feature_throughput * albedo`

## 依赖关系

- **内部头文件**:
  - `kernel/closure/bsdf.h` — 提供 `bsdf_albedo()`、`bsdf_get_specular_roughness_squared()` 等闭包/双向散射分布函数评估函数
  - `kernel/film/write.h` — 渲染通道写入基础设施

- **被引用**:
  - `src/kernel/integrator/shade_surface.h` — 表面着色时调用 `film_write_denoising_features_surface()`
  - `src/kernel/integrator/shade_volume.h` — 体积着色时调用 `film_write_denoising_features_volume()`

## 实现细节 / 关键算法

### 镜面反射延迟写入策略

这是本文件最核心的算法设计。降噪器需要的特征应该反映场景的"漫反射"属性，因为漫反射表面的信息最能帮助降噪器区分噪声和结构：

1. 遍历命中点上所有闭包/双向散射分布函数
2. 将粗糙度平方 > 0.075^2 的闭包归类为"非镜面"（漫反射类），其余归类为"镜面"
3. 黄氏毛发模型（远场）特殊处理：使用纤维切线方向而非法线，且计为漫反射类
4. 如果非镜面权重占比 > 25%（`sum_nonspecular_weight * 4 > sum_weight`），则当前弹射就写入降噪特征，并清除 `PATH_RAY_DENOISING_FEATURES` 标志
5. 否则，将镜面反射反照率乘入 `denoising_feature_throughput`，等待下一次弹射（可能命中漫反射表面）再写入

这种策略确保降噪法线和反照率主要反映漫反射特性，避免镜面高光区域的法线信息误导降噪器。

### 法线空间变换

降噪法线在写入前通过 `kernel_data.cam.worldtocamera` 矩阵从世界空间变换到相机空间。这是降噪器的标准要求，因为降噪通常在屏幕空间进行。

## 关联文件

- `src/kernel/film/write.h` — 渲染通道写入基础设施
- `src/kernel/film/light_passes.h` — 光照通道写入，也包含降噪反照率相关的辅助逻辑
- `src/kernel/closure/bsdf.h` — 闭包/双向散射分布函数评估
- `src/kernel/integrator/shade_surface.h` — 表面着色入口
- `src/kernel/integrator/shade_volume.h` — 体积着色入口
