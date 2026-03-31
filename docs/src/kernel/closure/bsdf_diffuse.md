# bsdf_diffuse.h - Lambertian 漫反射与半透明 BSDF

## 概述

`bsdf_diffuse.h` 实现了 Cycles 中最基础的两种漫反射闭包：Lambertian 漫反射（Diffuse）和半透明漫反射（Translucent）。Lambertian 漫反射假设光线在表面各方向均匀散射，是物理渲染中最常用的漫反射模型；半透明漫反射则在法线反面产生余弦加权分布的透射光，用于模拟薄叶片、纸张等半透明材质。

## 类与结构体

### DiffuseBsdf
```c
struct DiffuseBsdf {
    SHADER_CLOSURE_BASE;
};
```

最简单的闭包结构体，仅包含基类字段（`type`、`weight`、`sample_weight`、`N` 等），无额外参数。同时被漫反射和半透明两种闭包类型共用。

## 核心函数

### bsdf_diffuse_setup()
- **签名**: `ccl_device int bsdf_diffuse_setup(ccl_private DiffuseBsdf *bsdf)`
- **功能**: 初始化 Lambertian 漫反射闭包，设置类型为 `CLOSURE_BSDF_DIFFUSE_ID`。返回 `SD_BSDF | SD_BSDF_HAS_EVAL`。

### bsdf_diffuse_eval()
- **签名**: `ccl_device Spectrum bsdf_diffuse_eval(const ccl_private ShaderClosure *sc, const float3 wi, const float3 wo, ccl_private float *pdf)`
- **功能**: 计算 Lambertian 漫反射 BSDF 值。输出值和 PDF 均为 `max(N.wo, 0) / pi`，即余弦加权除以 pi 的归一化常数。入射方向 `wi` 未被使用（Lambertian 模型与入射方向无关）。

### bsdf_diffuse_sample()
- **签名**: `ccl_device int bsdf_diffuse_sample(const ccl_private ShaderClosure *sc, const float3 Ng, const float3 wi, const float2 rand, ccl_private Spectrum *eval, ccl_private float3 *wo, ccl_private float *pdf)`
- **功能**: 使用余弦加权半球采样（`sample_cos_hemisphere`）生成出射方向。验证采样方向位于几何法线正侧。返回 `LABEL_REFLECT | LABEL_DIFFUSE`。

### bsdf_translucent_setup()
- **签名**: `ccl_device int bsdf_translucent_setup(ccl_private DiffuseBsdf *bsdf)`
- **功能**: 初始化半透明闭包，设置类型为 `CLOSURE_BSDF_TRANSLUCENT_ID`。返回 `SD_BSDF | SD_BSDF_HAS_EVAL | SD_BSDF_HAS_TRANSMISSION`（额外标记具有透射特性）。

### bsdf_translucent_eval()
- **签名**: `ccl_device Spectrum bsdf_translucent_eval(const ccl_private ShaderClosure *sc, const float3 wi, const float3 wo, ccl_private float *pdf)`
- **功能**: 计算半透明漫反射值。与漫反射类似但使用法线负方向：`max(-N.wo, 0) / pi`，即在法线反面的半球内产生余弦分布。

### bsdf_translucent_sample()
- **签名**: `ccl_device int bsdf_translucent_sample(const ccl_private ShaderClosure *sc, const float3 Ng, const float3 wi, const float2 rand, ccl_private Spectrum *eval, ccl_private float3 *wo, ccl_private float *pdf)`
- **功能**: 在法线反方向的半球上进行余弦加权采样（`sample_cos_hemisphere(-N, ...)`）。验证采样方向位于几何法线反侧。返回 `LABEL_TRANSMIT | LABEL_DIFFUSE`。

## 依赖关系

- **内部头文件**:
  - `kernel/types.h` — 核心类型定义
  - `kernel/sample/mapping.h` — 提供 `sample_cos_hemisphere` 采样函数

- **被引用**:
  - `kernel/closure/bsdf.h` — 通过统一调度接口调用

## 实现细节 / 关键算法

### Lambertian 漫反射

经典的 Lambertian 模型：`f(wi, wo) = 1/pi`。由于余弦加权采样的 PDF 恰好等于 `cos(theta)/pi`，采样时 BSDF/PDF 的比率恰好等于 1，因此 `eval` 直接返回 `pdf` 值作为频谱值（`*eval = make_spectrum(*pdf)`）。

### 半透明模型

半透明模型的物理意义是光线穿过薄表面后在反侧产生漫反射分布。通过将法线翻转（`-N`）并在翻转后的半球上采样实现。验证方向时也使用翻转的逻辑：`dot(Ng, *wo) < 0` 表示出射方向在几何表面的反面。

## 关联文件

- `kernel/closure/bsdf.h` — 统一调度中心
- `kernel/sample/mapping.h` — 半球采样
- `kernel/svm/closure.h` — SVM 中创建漫反射/半透明闭包
