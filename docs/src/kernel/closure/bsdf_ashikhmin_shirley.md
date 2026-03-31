# bsdf_ashikhmin_shirley.h - Ashikhmin-Shirley 各向异性 Phong BRDF 模型

## 概述

`bsdf_ashikhmin_shirley.h` 实现了 Michael Ashikhmin 和 Peter Shirley 于 2000 年发表的论文 "An Anisotropic Phong BRDF Model" 中的各向异性光泽反射 BRDF。该模型支持各向异性和各向同性两种模式，适用于拉丝金属、丝绸等具有方向性反射特征的材质。实现中省略了菲涅尔（Fresnel）因子以获得可分离的 BSDF（强度 * 颜色），与 Cycles 中其他微面元（Microfacet）BSDF 的处理方式一致。

## 类与结构体

本文件复用 `bsdf_microfacet.h` 中定义的 `MicrofacetBsdf` 结构体，使用其中的以下字段：
- **`alpha_x`** / **`alpha_y`**: 各向异性粗糙度参数（范围 [1e-4, 1.0]）
- **`N`**: 着色法线
- **`T`**: 切线方向（用于各向异性时构建局部坐标系）
- **`fresnel_type`**: 设置为 `MicrofacetFresnel::NONE`

## 核心函数

### bsdf_ashikhmin_shirley_setup()
- **签名**: `ccl_device int bsdf_ashikhmin_shirley_setup(ccl_private MicrofacetBsdf *bsdf)`
- **功能**: 初始化闭包。将 `alpha_x` 和 `alpha_y` 限制在 [1e-4, 1.0] 范围内，设置闭包类型为 `CLOSURE_BSDF_ASHIKHMIN_SHIRLEY_ID`，返回 `SD_BSDF | SD_BSDF_HAS_EVAL` 标志。

### bsdf_ashikhmin_shirley_blur()
- **签名**: `ccl_device void bsdf_ashikhmin_shirley_blur(ccl_private ShaderClosure *sc, const float roughness)`
- **功能**: 将粗糙度参数提升到至少 `roughness` 值，用于 Filter Glossy 功能减少焦散噪点。

### bsdf_ashikhmin_shirley_roughness_to_exponent()
- **签名**: `ccl_device_inline float bsdf_ashikhmin_shirley_roughness_to_exponent(const float roughness)`
- **功能**: 将粗糙度参数转换为 Phong 指数：`e = 2 / roughness^2 - 2`。

### bsdf_ashikhmin_shirley_eval()
- **签名**: `ccl_device_forceinline Spectrum bsdf_ashikhmin_shirley_eval(const ccl_private ShaderClosure *sc, const float3 wi, const float3 wo, ccl_private float *pdf)`
- **功能**: 根据论文中的公式计算 BRDF 值和 PDF。支持各向同性（`n_x == n_y` 时使用简化公式）和各向异性（使用切线空间中的方向分量）两种路径。当粗糙度极低（<= 1e-4）或方向无效时返回零。

### bsdf_ashikhmin_shirley_sample_first_quadrant()
- **签名**: `ccl_device_inline void bsdf_ashikhmin_shirley_sample_first_quadrant(float n_x, const float n_y, const float2 rand, ccl_private float *phi, ccl_private float *cos_theta)`
- **功能**: 在第一象限内采样各向异性高光分布的球面坐标。通过 atan 和 pow 函数根据各向异性指数进行重要性采样。

### bsdf_ashikhmin_shirley_sample()
- **签名**: `ccl_device int bsdf_ashikhmin_shirley_sample(const ccl_private ShaderClosure *sc, const float3 Ng, const float3 wi, float2 rand, ccl_private Spectrum *eval, ccl_private float3 *wo, ccl_private float *pdf, ccl_private float2 *sampled_roughness)`
- **功能**: 采样出射方向。各向同性模式使用简单的幂分布采样；各向异性模式将随机数映射到四个象限分别采样后组合。采样半向量后通过反射得到出射方向。粗糙度极低时视为奇异反射（SINGULAR），返回高 PDF 值用于 MIS（多重重要性采样）。

## 依赖关系

- **内部头文件**:
  - `kernel/types.h` — 核心类型定义
  - `kernel/closure/bsdf_microfacet.h` — 提供 `MicrofacetBsdf` 结构体定义

- **被引用**:
  - `kernel/closure/bsdf.h` — 通过统一调度接口调用

## 实现细节 / 关键算法

### Phong BRDF 公式

BRDF 表达式遵循论文公式，核心计算为：
- **pump 因子**: `1 / (H.I * max(N.I, N.O))` — 源自原始论文的一阶导数不连续性修正
- **各向同性 lobe**: `(N.H)^e`，归一化因子为 `(e+1)/(8*pi)`
- **各向异性 lobe**: `(N.H)^((n_x * H.X^2 + n_y * H.Y^2) / (1 - N.H^2))`，归一化因子使用两个指数的几何平均

### 各向异性采样的象限策略

对各向异性情况，随机数 `rand.x` 被映射到四个象限：
- [0, 0.25): 第一象限，直接采样
- [0.25, 0.5): 第二象限，`phi = pi - phi`
- [0.5, 0.75): 第三象限，`phi = pi + phi`
- [0.75, 1.0): 第四象限，`phi = 2*pi - phi`

这种方式利用了 Phong 分布的对称性，同时保持了各向异性特征。

## 关联文件

- `kernel/closure/bsdf_microfacet.h` — 提供 `MicrofacetBsdf` 结构体
- `kernel/closure/bsdf.h` — 统一调度中心，调用 `bsdf_ashikhmin_shirley_sample` / `bsdf_ashikhmin_shirley_eval` / `bsdf_ashikhmin_shirley_blur`
