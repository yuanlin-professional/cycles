# bsdf_hair.h - 基础毛发反射/透射 BSDF 模型

## 概述

`bsdf_hair.h` 实现了 Cycles 中基础的毛发着色器（Hair BSDF），提供毛发反射（Hair Reflection）和毛发透射（Hair Transmission）两种闭包类型。该模型使用沿毛发切线方向的柯西（Cauchy/Lorentzian）分布来描述纵向和方位角散射，通过 `roughness1` 和 `roughness2` 两个参数独立控制。此实现适用于基本毛发渲染，为 Principled Hair 提供较简单的替代方案。

## 类与结构体

### HairBsdf
```c
struct HairBsdf {
    SHADER_CLOSURE_BASE;
    float3 T;           // 毛发切线方向
    float roughness1;   // 纵向散射粗糙度，范围 [0.001, 1.0]
    float roughness2;   // 方位角散射粗糙度，范围 [0.001, 1.0]
    float offset;       // 表皮倾斜偏移角度
};
```

## 核心函数

### bsdf_hair_reflection_setup()
- **签名**: `ccl_device int bsdf_hair_reflection_setup(ccl_private HairBsdf *bsdf)`
- **功能**: 初始化毛发反射闭包。将粗糙度限制在 [0.001, 1.0]，设置类型为 `CLOSURE_BSDF_HAIR_REFLECTION_ID`。返回 `SD_BSDF | SD_BSDF_HAS_EVAL`。

### bsdf_hair_transmission_setup()
- **签名**: `ccl_device int bsdf_hair_transmission_setup(ccl_private HairBsdf *bsdf)`
- **功能**: 初始化毛发透射闭包。与反射类似但设置类型为 `CLOSURE_BSDF_HAIR_TRANSMISSION_ID`，额外返回 `SD_BSDF_HAS_TRANSMISSION` 标志。

### bsdf_hair_reflection_eval()
- **签名**: `ccl_device Spectrum bsdf_hair_reflection_eval(const ccl_private ShaderClosure *sc, const float3 wi, const float3 wo, ccl_private float *pdf)`
- **功能**: 计算毛发反射 BSDF 值。将入射/出射方向投影到切线空间，计算纵向角 `theta` 和方位角 `phi`。纵向使用柯西分布，方位角使用余弦分布。仅在出射方向位于法线正侧且 `cosphi_i >= 0` 时有效。

### bsdf_hair_transmission_eval()
- **签名**: `ccl_device Spectrum bsdf_hair_transmission_eval(const ccl_private ShaderClosure *sc, const float3 wi, const float3 wo, ccl_private float *pdf)`
- **功能**: 计算毛发透射 BSDF 值。与反射类似但出射方向应在法线反侧，方位角分布使用 `phi = pi - phi` 的反向柯西分布（集中在与入射方向相反的方位角区域）。

### bsdf_hair_reflection_sample()
- **签名**: `ccl_device int bsdf_hair_reflection_sample(const ccl_private ShaderClosure *sc, const float3 Ng, const float3 wi, const float2 rand, ccl_private Spectrum *eval, ccl_private float3 *wo, ccl_private float *pdf, ccl_private float2 *sampled_roughness)`
- **功能**: 反射方向的重要性采样。通过对柯西分布的 CDF 逆变换采样纵向角（使用 `tan` 变换），方位角使用 `safe_asinf` 变换。返回 `LABEL_REFLECT | LABEL_GLOSSY`。

### bsdf_hair_transmission_sample()
- **签名**: `ccl_device int bsdf_hair_transmission_sample(const ccl_private ShaderClosure *sc, const float3 Ng, const float3 wi, const float2 rand, ccl_private Spectrum *eval, ccl_private float3 *wo, ccl_private float *pdf, ccl_private float2 *sampled_roughness)`
- **功能**: 透射方向的重要性采样。类似反射但方位角偏移 pi 以产生前向散射。返回 `LABEL_TRANSMIT | LABEL_GLOSSY`。

## 依赖关系

- **内部头文件**:
  - `kernel/types.h` — 核心类型定义
  - `util/math_fast.h` — 快速三角函数（`fast_acosf`、`fast_atan2f`、`fast_cosf`、`fast_sincosf`）

- **被引用**:
  - `kernel/closure/bsdf.h` — 通过统一调度接口调用

## 实现细节 / 关键算法

### 切线空间投影

毛发 BSDF 在切线空间中工作：
1. 将入射方向 `wi` 投影到切线 `T` 上得到纵向分量 `Iz = dot(T, wi)`
2. 纵向角 `theta_r = pi/2 - acos(Iz)`
3. 方位角通过入射方向在垂直于切线的平面上的投影 `locy` 和出射方向的投影 `wo_y` 的点积得到

### 纵向散射（Cauchy/Lorentzian 分布）

```
theta_pdf = roughness / (2 * (t^2 + roughness^2) * (a - b) * cos(theta_i))
```

其中 `t = theta_h - offset`（半角与偏移角的差），`a` 和 `b` 是积分归一化区间的 arctan 值。采样使用柯西分布的逆 CDF：`t = roughness * tan(u * (a - b) + b)`。

### 方位角散射

- **反射**：`phi_pdf = cos(phi/2) * 0.25 / roughness2`
- **透射**：`phi_pdf = roughness2 / (c * (p^2 + roughness2^2))`，其中 `p = pi - |phi|`

## 关联文件

- `kernel/closure/bsdf.h` — 统一调度中心
- `kernel/closure/bsdf_principled_hair_chiang.h` — 更高级的 Principled Hair 模型（Chiang 2015）
- `kernel/closure/bsdf_principled_hair_huang.h` — 基于微面元的 Principled Hair 模型（Huang 2022）
