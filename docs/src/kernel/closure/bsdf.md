# bsdf.h - BSDF 统一调度与求值入口

## 概述

`bsdf.h` 是 Cycles 渲染内核中所有双向散射分布函数（BSDF）的统一调度中心。它通过包含所有具体 BSDF 实现的头文件，并提供 `bsdf_sample`、`bsdf_eval`、`bsdf_label` 等入口函数，根据闭包类型（`ClosureType`）将请求分发到对应的 BSDF 实现。此文件还包含凹凸贴图阴影修正（bump shadowing term）和阴影终结器偏移（shadow terminator offset）等通用辅助功能。

## 类与结构体

本文件不定义新的结构体，但引用所有 BSDF 实现中定义的结构体（如 `MicrofacetBsdf`、`OrenNayarBsdf`、`HairBsdf` 等）。

## 核心函数

### bsdf_get_specular_roughness_squared()
- **签名**: `ccl_device_inline float bsdf_get_specular_roughness_squared(const ccl_private ShaderClosure *sc)`
- **功能**: 返回闭包的镜面粗糙度平方值。对于奇异闭包（singular）返回 0，对于微面元（Microfacet）类型返回 `alpha_x * alpha_y`，其他类型返回 1。

### bsdf_get_roughness_pass_squared()
- **签名**: `ccl_device_inline float bsdf_get_roughness_pass_squared(const ccl_private ShaderClosure *sc)`
- **功能**: 用于粗糙度渲染通道的输出。对于 Oren-Nayar 返回其粗糙度的四次方，对于漫反射类型返回 -1（表示不适用），其他类型委托给 `bsdf_get_specular_roughness_squared`。

### bump_shadowing_term()
- **签名**: `ccl_device_inline float bump_shadowing_term(const ccl_private ShaderData *sd, const ccl_private ShaderClosure *sc, const float3 I, const bool is_eval)`
- **功能**: 基于论文 "A Microfacet-Based Shadowing Function to Solve the Bump Terminator Problem"（Conty Estevez 等人）实现的凹凸贴图阴影修正项。通过比较着色法线 `sc->N` 与平滑法线 `sd->N` 的关系，防止凹凸贴图导致的光线泄漏到几何体背面。对非漫反射闭包仅做方向有效性检查；对漫反射闭包额外施加基于 GGX分布的平滑过渡。

### shift_cos_in()
- **签名**: `ccl_device_inline float shift_cos_in(float cos_in, const float frequency_multiplier)`
- **功能**: 阴影终结器偏移的辅助函数，源自 Appleseed 渲染器。通过频率倍增器调整入射余弦值，平滑阴影终结处的明暗过渡。

### bsdf_is_transmission()
- **签名**: `ccl_device_inline bool bsdf_is_transmission(const ccl_private ShaderClosure *sc, const float3 wo)`
- **功能**: 判断给定出射方向 `wo` 相对于闭包法线是否为透射方向。

### bsdf_sample()
- **签名**: `ccl_device_inline int bsdf_sample(KernelGlobals kg, ccl_private ShaderData *sd, const ccl_private ShaderClosure *sc, const float3 rand, ccl_private Spectrum *eval, ccl_private float3 *wo, ccl_private float *pdf, ccl_private float2 *sampled_roughness, ccl_private float *eta)`
- **功能**: BSDF 统一采样入口。根据闭包类型通过 `switch` 语句分发到对应的具体 BSDF 采样函数。完成采样后，对透射方向进行背景透明度粗糙度阈值检查，对反射方向施加阴影终结器偏移和凹凸阴影修正。返回路径标签（`LABEL_REFLECT`、`LABEL_TRANSMIT`、`LABEL_DIFFUSE`、`LABEL_GLOSSY` 等组合）。

### bsdf_roughness_eta()
- **签名**: `ccl_device_inline void bsdf_roughness_eta(const ccl_private ShaderClosure *sc, const float3 wo, ccl_private float2 *roughness, ccl_private float *eta)`
- **功能**: 返回给定闭包的粗糙度和折射率（eta），用于引导路径采样和降噪等辅助系统。

### bsdf_label()
- **签名**: `ccl_device_inline int bsdf_label(const KernelGlobals kg, const ccl_private ShaderClosure *sc, const float3 wo)`
- **功能**: 根据闭包类型和出射方向确定路径标签。用于光线分类（如漫反射、光泽反射、透射、透明等），对于微面元类型还会根据粗糙度判断是光泽（GLOSSY）还是奇异（SINGULAR）。

### bsdf_eval()
- **签名**: `ccl_device Spectrum bsdf_eval(KernelGlobals kg, ccl_private ShaderData *sd, const ccl_private ShaderClosure *sc, const float3 wo, ccl_private float *pdf)`
- **功能**: BSDF 统一求值入口。根据闭包类型分发到对应的 BSDF 求值函数。先检查凹凸阴影项，若为零直接返回；求值后乘以凹凸阴影修正和阴影终结器偏移。

### bsdf_blur()
- **签名**: `ccl_device void bsdf_blur(ccl_private ShaderClosure *sc, const float roughness)`
- **功能**: 对闭包施加粗糙度模糊（Filter Glossy），用于减少焦散噪点。仅对支持此操作的闭包类型生效（微面元类型、Ashikhmin-Shirley、Principled Hair）。

### bsdf_albedo()
- **签名**: `ccl_device_inline Spectrum bsdf_albedo(KernelGlobals kg, const ccl_private ShaderData *sd, const ccl_private ShaderClosure *sc, const bool reflection, const bool transmission)`
- **功能**: 返回闭包的反照率估计，用于降噪反照率通道和漫反射/光泽/透射颜色通道。对微面元类型和 Principled Hair 类型应用额外的菲涅尔（Fresnel）修正。

## 依赖关系

- **内部头文件**:
  - `kernel/closure/bsdf_ashikhmin_velvet.h`
  - `kernel/closure/bsdf_diffuse.h`
  - `kernel/closure/bsdf_oren_nayar.h`
  - `kernel/closure/bsdf_phong_ramp.h`
  - `kernel/closure/bsdf_diffuse_ramp.h`
  - `kernel/closure/bsdf_microfacet.h`
  - `kernel/closure/bsdf_burley.h`
  - `kernel/closure/bsdf_sheen.h`
  - `kernel/closure/bsdf_transparent.h`
  - `kernel/closure/bsdf_ray_portal.h`
  - `kernel/closure/bsdf_ashikhmin_shirley.h`
  - `kernel/closure/bsdf_toon.h`
  - `kernel/closure/bsdf_hair.h`
  - `kernel/closure/bsdf_principled_hair_chiang.h`
  - `kernel/closure/bsdf_principled_hair_huang.h`

- **被引用**:
  - `kernel/integrator/surface_shader.h` — 表面着色器积分
  - `kernel/integrator/guiding.h` — 路径引导
  - `kernel/film/denoising_passes.h` — 降噪通道
  - `kernel/svm/closure.h` — SVM 闭包系统
  - `kernel/osl/closures_setup.h` — OSL 闭包系统

## 实现细节 / 关键算法

### 凹凸终结器修正（Bump Terminator Fix）

`bump_shadowing_term` 解决了凹凸/法线贴图导致的光线泄漏问题：
1. 首先检查入射方向 `I` 和着色法线 `N` 相对于平滑法线 `Ns` 的同侧/异侧关系
2. 对于漫反射闭包，使用 GGX分布的遮蔽函数进行平滑过渡
3. 对于曲线几何体（curves）跳过此修正

### 阴影终结器偏移（Shadow Terminator Offset）

在采样和求值后，对反射光线应用基于 Appleseed 方法的阴影终结器偏移。通过 `frequency_multiplier`（从物体属性中获取）压缩 cos 值的变化频率，使阴影边界更加平滑。

### 透明背景粗糙度阈值

对透射方向的光线，如果闭包粗糙度低于全局设置的阈值 `transparent_roughness_squared_threshold`，则将其标记为 `LABEL_TRANSMIT_TRANSPARENT`，使其被视为透明背景射线。

## 关联文件

- `kernel/integrator/surface_shader.h` — 调用 `bsdf_sample` 和 `bsdf_eval`
- `kernel/integrator/guiding.h` — 调用 `bsdf_roughness_eta` 进行路径引导
- `kernel/film/denoising_passes.h` — 调用 `bsdf_albedo` 生成降噪通道
