# bsdf_microfacet.h - 微面元双向散射分布函数（BSDF）核心实现

## 概述

本文件实现了 Cycles 渲染器中基于微面元（Microfacet）理论的双向散射分布函数（BSDF），是整个材质系统中最核心的闭包（Closure）之一。文件支持 GGX 和 Beckmann 两种法线分布函数（NDF），并提供了包含电介质、导体、广义 Schlick、F82 色调等多种菲涅尔（Fresnel）模型的完整实现。此外，文件还包含基于 Kulla-Conty 方法的多重散射能量补偿机制，确保渲染的能量守恒。

## 类与结构体

### `enum MicrofacetType`
微面元分布类型枚举：
- `BECKMANN` — Beckmann 分布
- `GGX` — GGX 分布（Trowbridge-Reitz）

### `enum MicrofacetFresnel`
菲涅尔模型类型枚举：
- `NONE` — 无菲涅尔效果
- `DIELECTRIC` — 电介质菲涅尔
- `DIELECTRIC_TINT` — 带色调的电介质菲涅尔（用于 OSL MaterialX 闭包）
- `CONDUCTOR` — 导体菲涅尔
- `GENERALIZED_SCHLICK` — 广义 Schlick 菲涅尔
- `F82_TINT` — F82 色调模型（Adobe Standard Material）

### `struct FresnelDielectricTint`
带色调的电介质菲涅尔数据，包含薄膜干涉参数（`FresnelThinFilm thin_film`）、反射色调（`reflection_tint`）和透射色调（`transmission_tint`）。

### `struct FresnelConductor`
导体菲涅尔数据，包含薄膜干涉参数和复数折射率（`complex<Spectrum> ior`）。

### `struct FresnelGeneralizedSchlick`
广义 Schlick 菲涅尔数据，包含薄膜干涉参数、反射/透射色调、垂直入射反射率 `f0`、掠射角反射率 `f90` 以及指数参数 `exponent`（负值表示使用真实菲涅尔曲线重映射到 F0...F90）。

### `struct FresnelF82Tint`
F82 色调菲涅尔数据，包含薄膜干涉参数、垂直反射率 `f0` 和预计算的 `(1-cos)^6` 边缘色调因子 `b`。

### `struct MicrofacetBsdf`
微面元 BSDF 核心数据结构，继承自 `SHADER_CLOSURE_BASE`：
- `alpha_x`, `alpha_y` — X/Y 方向粗糙度参数
- `ior` — 折射率
- `energy_scale` — 多重散射能量补偿因子（仅 GGX 使用）
- `fresnel_type` — 菲涅尔模型类型
- `fresnel` — 指向具体菲涅尔数据的指针
- `T` — 各向异性切线方向

## 核心函数

### VNDF 重要性采样
- **`microfacet_beckmann_sample_vndf(wi, alpha_x, alpha_y, rand)`** — Beckmann 分布的可见法线分布函数（VNDF）采样，基于 Heitz & d'Eon (EGSR 2014) 的算法。
- **`microfacet_ggx_sample_vndf(wi, alpha_x, alpha_y, rand)`** — GGX 分布的 VNDF 采样，基于 Heitz (JCGT 2018) 的算法。

### 菲涅尔计算
- **`microfacet_fresnel(kg, bsdf, cos_theta_i, r_cos_theta_t, r_reflectance, r_transmittance)`** — 根据 BSDF 设定的菲涅尔类型计算反射率和透射率，支持薄膜干涉。

### 能量守恒
- **`microfacet_ggx_preserve_energy(kg, bsdf, sd, Fss)`** — 基于 Kulla-Conty 方法的多重散射能量补偿。通过查找表（LUT）获取单次散射能量 `E` 和平均能量 `E_avg`，计算缺失能量的补偿因子。

### 反照率估算
- **`bsdf_microfacet_estimate_albedo(kg, sd, bsdf, eval_reflection, eval_transmission)`** — 估算 BSDF 的反照率，用于调整采样权重和去噪反照率 Pass。对广义 Schlick 和 F82 色调模型使用预计算查找表。

### Smith 遮蔽-阴影项
- **`bsdf_lambda<m_type>(alpha2, cos_N)`** — 计算 Smith Lambda 函数。
- **`bsdf_aniso_lambda<m_type>(alpha_x, alpha_y, V)`** — 各向异性 Smith Lambda。
- **`bsdf_G<m_type>(alpha2, cos_N)`** — 单方向遮蔽项。
- **`bsdf_G<m_type>(alpha2, cos_NI, cos_NO)`** — 联合遮蔽-阴影项。

### 法线分布函数（NDF）
- **`bsdf_D<m_type>(alpha2, cos_NH)`** — 各向同性法线分布函数。
- **`bsdf_aniso_D<m_type>(alpha_x, alpha_y, H)`** — 各向异性法线分布函数。

### 求值与采样
- **`bsdf_microfacet_eval<m_type>(kg, sc, wi, wo, pdf)`** — 模板化的微面元 BSDF 求值函数，支持反射和折射，各向同性和各向异性。
- **`bsdf_microfacet_sample<m_type>(kg, sc, Ng, wi, rand, eval, wo, pdf, sampled_roughness, eta)`** — 模板化的微面元 BSDF 采样函数，根据反射率/透射率能量比选择反射或折射。

### GGX 特化接口
- **`bsdf_microfacet_ggx_setup(bsdf)`** — GGX 反射设置。
- **`bsdf_microfacet_ggx_refraction_setup(bsdf)`** — GGX 折射设置。
- **`bsdf_microfacet_ggx_glass_setup(bsdf)`** — GGX 玻璃（反射+折射）设置。
- **`bsdf_microfacet_ggx_eval(kg, sc, wi, wo, pdf)`** — GGX 求值（乘以 energy_scale）。
- **`bsdf_microfacet_ggx_sample(...)`** — GGX 采样（乘以 energy_scale）。

### Beckmann 特化接口
- **`bsdf_microfacet_beckmann_setup(bsdf)`** — Beckmann 反射设置。
- **`bsdf_microfacet_beckmann_refraction_setup(bsdf)`** — Beckmann 折射设置。
- **`bsdf_microfacet_beckmann_glass_setup(bsdf)`** — Beckmann 玻璃设置。
- **`bsdf_microfacet_beckmann_eval(kg, sc, wi, wo, pdf)`** — Beckmann 求值。
- **`bsdf_microfacet_beckmann_sample(...)`** — Beckmann 采样。

### 菲涅尔设置函数
- **`bsdf_microfacet_setup_fresnel_conductor(...)`** — 设置导体菲涅尔。
- **`bsdf_microfacet_setup_fresnel_dielectric_tint(...)`** — 设置带色调电介质菲涅尔。
- **`bsdf_microfacet_setup_fresnel_generalized_schlick(...)`** — 设置广义 Schlick 菲涅尔。
- **`bsdf_microfacet_setup_fresnel_f82_tint(...)`** — 设置 F82 色调菲涅尔。
- **`bsdf_microfacet_setup_fresnel_constant(...)`** — 设置常量菲涅尔（颜色已烘焙进权重）。
- **`bsdf_microfacet_setup_fresnel_dielectric(...)`** — 设置基本电介质菲涅尔。

### 工具函数
- **`bsdf_microfacet_blur(sc, roughness)`** — 实现 Filter Glossy，将粗糙度提升到给定最小值。
- **`bsdf_microfacet_eval_flag(bsdf)`** — 当粗糙度平方低于阈值时不设置 `SD_BSDF_HAS_EVAL` 标志（视为镜面）。

## 依赖关系
- **内部头文件**:
  - `kernel/closure/bsdf_util.h` — BSDF 工具函数（菲涅尔公式等）
  - `kernel/sample/mapping.h` — 采样映射工具
  - `kernel/util/lookup_table.h` — 查找表读取
  - `util/math_fast.h` — 快速数学函数（erf、ierf 等）
- **被引用**:
  - `kernel/closure/bsdf.h` — 闭包统一调度入口
  - `kernel/closure/bsdf_ashikhmin_shirley.h` — Ashikhmin-Shirley BSDF
  - `kernel/closure/bsdf_principled_hair_huang.h` — Huang 毛发模型
  - `app/cycles_precompute.cpp` — LUT 预计算工具

## 实现细节 / 关键算法

### 微面元理论框架
整体实现基于 Walter et al. (EGSR 2007) 的 Cook-Torrance 微面元框架，BSDF 公式为：
```
f(wi, wo) = F(wi, H) * D(H) * G(wi, wo) / (4 * cos(N, wi) * cos(N, wo))
```
其中 `F` 为菲涅尔项，`D` 为法线分布函数，`G` 为几何遮蔽-阴影项。

### VNDF 重要性采样
- GGX 采用 Heitz (JCGT 2018) 的球面帽采样方法。
- Beckmann 采用 Heitz & d'Eon (EGSR 2014) 的方法，辅以 Jakob 的改进可见法线采样。

### 多重散射能量补偿
基于 Kulla & Conty (SIGGRAPH 2017) 的方法，通过预计算查找表存储单次散射能量 `E(roughness, mu)` 和平均能量 `E_avg(roughness)`。对于玻璃材质使用三维查找表（额外维度为折射率）。补偿因子 `energy_scale = 1 + (1 - E) / E` 乘以 BSDF 值。

### 菲涅尔模型
- **广义 Schlick**: 支持自定义指数，当指数为负时使用真实电介质菲涅尔重映射到 F0...F90 范围（Principled BSDF 使用此模式）。
- **F82 色调**: 基于 Adobe Standard Material 中描述的模型，在约 82 度入射角处提供额外的色调调制。
- **薄膜干涉**: 所有菲涅尔模型均支持薄膜干涉效果，厚度超过 `THINFILM_THICKNESS_CUTOFF` 时启用。

### 各向异性支持
反射模式支持各向异性（`alpha_x != alpha_y`），通过切线方向 `T` 构建局部坐标系。折射目前仅支持各向同性。

## 关联文件
- `kernel/closure/bsdf_util.h` — 菲涅尔公式、反射/折射角度计算
- `kernel/closure/bsdf.h` — 闭包统一调度，根据闭包类型调用对应的 eval/sample
- `kernel/closure/bsdf_ashikhmin_shirley.h` — 复用微面元工具函数
- `kernel/closure/bsdf_principled_hair_huang.h` — 复用 GGX 分布和遮蔽函数
- `app/cycles_precompute.cpp` — 预计算能量补偿所需的查找表
