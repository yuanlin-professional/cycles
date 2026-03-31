# bsdf_burley.h - Burley 漫反射 BSDF 模型

## 概述

`bsdf_burley.h` 实现了 Brent Burley 提出的基于物理的漫反射模型（常称为 Disney Diffuse）。该模型在标准 Lambertian 漫反射基础上引入了粗糙度相关的菲涅尔（Fresnel）调制项，使粗糙表面在掠射角处呈现逆反射增亮效果。此闭包仅在 OSL（Open Shading Language）模式下可用。

## 类与结构体

### BurleyBsdf
```c
struct BurleyBsdf {
    SHADER_CLOSURE_BASE;
    float roughness;    // 表面粗糙度，范围 [0, 1]
};
```

继承自 `SHADER_CLOSURE_BASE`，仅包含一个额外参数 `roughness`。通过 `static_assert` 确保其大小不超过 `ShaderClosure`。

## 核心函数

### bsdf_burley_get_intensity()
- **签名**: `ccl_device Spectrum bsdf_burley_get_intensity(const float roughness, const float3 n, const float3 v, const float3 l)`
- **功能**: 计算 Burley 漫反射的强度。使用 Schlick 近似计算入射和出射方向的菲涅尔项 `fl` 和 `fv`，结合粗糙度相关的 F90 因子 `F90 = 0.5 + 2 * roughness * (L.H)^2`。最终输出为 `(1/pi) * N.L * mix(1, F90, fl) * mix(1, F90, fv)`。

### bsdf_burley_setup()
- **签名**: `ccl_device int bsdf_burley_setup(ccl_private BurleyBsdf *bsdf, const float roughness)`
- **功能**: 初始化 Burley 漫反射闭包。将粗糙度限制到 [0, 1] 范围（`saturatef`），设置闭包类型为 `CLOSURE_BSDF_BURLEY_ID`。返回 `SD_BSDF | SD_BSDF_HAS_EVAL` 标志。

### bsdf_burley_eval()
- **签名**: `ccl_device Spectrum bsdf_burley_eval(ccl_private const ShaderClosure *sc, const float3 wi, const float3 wo, ccl_private float *pdf)`
- **功能**: 求值函数。仅在 `cos(N, wo) > 0`（上半球）时有效，PDF 使用余弦加权半球分布 `cos(N, wo) / pi`，BSDF 值由 `bsdf_burley_get_intensity` 计算。

### bsdf_burley_sample()
- **签名**: `ccl_device int bsdf_burley_sample(ccl_private const ShaderClosure *sc, float3 Ng, float3 wi, float2 rand, ccl_private Spectrum *eval, ccl_private float3 *wo, ccl_private float *pdf)`
- **功能**: 使用余弦加权半球采样（`sample_cos_hemisphere`）生成出射方向。验证采样方向在几何法线正侧后计算 BSDF 值。返回标签 `LABEL_REFLECT | LABEL_DIFFUSE`。

## 依赖关系

- **内部头文件**:
  - `kernel/closure/bsdf_util.h` — 提供 `schlick_fresnel` 等菲涅尔工具函数
  - `kernel/sample/mapping.h` — 提供 `sample_cos_hemisphere` 采样函数

- **被引用**:
  - `kernel/closure/bsdf.h` — 通过统一调度接口调用（在 `__OSL__` 宏条件下）

## 实现细节 / 关键算法

### Burley 漫反射公式

Burley 模型通过两个 Schlick 菲涅尔项修正 Lambertian 漫反射：

```
f(wi, wo) = (1/pi) * (N.L) * (1 + (F90 - 1) * (1 - N.L)^5) * (1 + (F90 - 1) * (1 - N.V)^5)
```

其中 `F90 = 0.5 + 2 * roughness * (L.H)^2`。

- **roughness = 0** 时：F90 = 0.5，在掠射角有轻微变暗
- **roughness = 1** 时：F90 最大可达 2.5，产生显著的掠射角增亮效果

### 条件编译

整个实现被 `#ifdef __OSL__` 包裹，仅在 OSL 模式下编译。这是因为 Burley 漫反射主要由 OSL 着色器使用；SVM 着色器系统使用 Principled BSDF 的漫反射组件来实现类似效果。

## 关联文件

- `kernel/closure/bsdf_util.h` — 菲涅尔工具函数
- `kernel/closure/bsdf.h` — 统一调度中心
- `kernel/osl/closures_setup.h` — OSL 闭包创建
