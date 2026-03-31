# closure.h - 着色器闭包节点的SVM实现

## 概述

`closure.h` 是 Cycles 着色器虚拟机(SVM)中最核心、最庞大的文件之一，负责实现所有闭包(Closure)节点的内核逻辑。该文件涵盖了表面双向散射分布函数(BSDF)、体积散射/吸收、自发光、背景、Holdout 以及闭包混合等节点的创建和参数设置。它是将着色器节点图中的闭包节点转化为实际光线传输计算的关键桥梁。

## 核心函数

### `svm_node_closure_bsdf_skip`
- **签名**: `ccl_device_inline int svm_node_closure_bsdf_skip(KernelGlobals kg, int offset, const uint type)`
- **功能**: 跳过指定闭包类型的额外数据节点，用于在不需要计算闭包时正确推进字节码偏移量。对于 Principled BSDF 会额外跳过两个节点。

### `svm_node_closure_bsdf`
- **签名**: `template<uint node_feature_mask, ShaderType shader_type> int svm_node_closure_bsdf(KernelGlobals kg, ShaderData *sd, float *stack, Spectrum closure_weight, const uint4 node, const uint32_t path_flag, int offset)`
- **功能**: 主要的表面 BSDF 闭包分发函数。根据闭包类型(通过 `switch` 语句)分配并初始化对应的 BSDF 结构体。支持的类型包括：
  - `CLOSURE_BSDF_PRINCIPLED_ID` — Principled BSDF（最复杂，含光泽层、涂层、金属、透射、次表面散射、漫反射等分层结构）
  - `CLOSURE_BSDF_DIFFUSE_ID` — 漫反射 / Oren-Nayar
  - `CLOSURE_BSDF_TRANSLUCENT_ID` — 半透明漫反射
  - `CLOSURE_BSDF_TRANSPARENT_ID` — 完全透明
  - `CLOSURE_BSDF_PHYSICAL_CONDUCTOR` / `CLOSURE_BSDF_F82_CONDUCTOR` — 物理导体/F82色调导体
  - `CLOSURE_BSDF_RAY_PORTAL_ID` — 光线传送门
  - `CLOSURE_BSDF_MICROFACET_GGX_ID` / `BECKMANN` / `ASHIKHMIN_SHIRLEY` / `MULTI_GGX` — 微面元反射
  - `CLOSURE_BSDF_MICROFACET_*_REFRACTION_ID` — 微面元折射
  - `CLOSURE_BSDF_MICROFACET_*_GLASS_ID` — 微面元玻璃（反射+折射）
  - `CLOSURE_BSDF_ASHIKHMIN_VELVET_ID` — 天鹅绒
  - `CLOSURE_BSDF_SHEEN_ID` — 光泽
  - `CLOSURE_BSDF_*_TOON_ID` — 卡通着色
  - `CLOSURE_BSDF_HAIR_*` — 毛发 BSDF（Chiang/Huang 模型）
  - `CLOSURE_BSSRDF_*` — 次表面散射（Burley/Random Walk）

### `svm_alloc_closure_volume_scatter`
- **签名**: `ccl_device_inline void svm_alloc_closure_volume_scatter(...)`
- **功能**: 分配体积散射闭包，支持 Henyey-Greenstein、Fournier-Forand、Rayleigh、Draine 和 Mie 等相函数类型。

### `svm_node_closure_volume`
- **签名**: `template<ShaderType shader_type> void svm_node_closure_volume(...)`
- **功能**: 体积着色器的闭包节点，计算体积的散射系数和消光权重。

### `svm_node_volume_coefficients`
- **签名**: `template<ShaderType shader_type> void svm_node_volume_coefficients(...)`
- **功能**: 处理体积系数节点，设置散射、吸收和自发光系数。

### `svm_node_principled_volume`
- **签名**: `template<ShaderType shader_type> int svm_node_principled_volume(...)`
- **功能**: Principled Volume 节点实现。计算密度（支持属性查找）、散射颜色、吸收、自发光和黑体辐射（Stefan-Boltzmann 定律）。

### `svm_node_closure_emission`
- **功能**: 自发光闭包节点，设置表面或体积的自发光。

### `svm_node_closure_background`
- **功能**: 背景闭包节点，用于世界环境着色器。

### `svm_node_closure_holdout`
- **功能**: Holdout 闭包节点，标记区域为遮挡区域（设置 `SD_HOLDOUT` 标志）。

### `svm_node_closure_set_weight` / `svm_node_closure_weight`
- **功能**: 设置闭包权重，从常量或栈中读取 RGB 权重值并转换为光谱。

### `svm_node_emission_weight`
- **功能**: 设置自发光权重（颜色 * 强度）。

### `svm_node_mix_closure`
- **功能**: 混合闭包节点，根据混合因子计算两个闭包的权重比例。

### `svm_node_set_normal`
- **功能**: 设置着色法线（用于凹凸贴图等）。

## 依赖关系

- **内部头文件**:
  - `kernel/closure/alloc.h` — 闭包内存分配
  - `kernel/closure/bsdf.h` — BSDF 闭包定义与设置
  - `kernel/closure/bsdf_util.h` — BSDF 工具函数
  - `kernel/closure/bssrdf.h` — 次表面散射
  - `kernel/closure/emissive.h` — 自发光闭包
  - `kernel/closure/volume.h` — 体积闭包
  - `kernel/geom/curve.h` — 曲线几何
  - `kernel/geom/object.h` — 物体变换
  - `kernel/geom/primitive.h` — 图元属性
  - `kernel/svm/math_util.h` — 数学工具（含黑体颜色计算）
  - `kernel/svm/util.h` — SVM 栈操作工具
  - `kernel/util/colorspace.h` — 色彩空间转换
- **被引用**: `kernel/svm/svm.h`（SVM 主调度文件）

## 实现细节 / 关键算法

1. **Principled BSDF 分层架构**: 按照从外到内的顺序依次处理各层：光泽层(Sheen) -> 涂层(Coat) -> 自发光(Emission) -> 金属(Metallic) -> 透射(Transmission) -> 镜面反射(Specular) -> 漫反射/次表面散射(Diffuse/SSS)。每层处理后通过 `closure_layering_weight` 计算衰减权重，传递给下层。

2. **焦散技巧(Caustics Tricks)**: 通过 `__CAUSTICS_TRICKS__` 宏控制，在漫反射光线路径上可选择性地跳过镜面反射/折射闭包，以减少噪点。

3. **涂层色调(Coat Tint)**: 使用 Beer 定律计算涂层吸收，通过折射角度计算光学深度 `1/cosNT`，再应用 `tint^optical_depth` 得到衰减。

4. **薄膜干涉(Thin Film)**: 在 Principled BSDF、导体和玻璃闭包中支持薄膜干涉效果，通过 `thin_film.thickness` 和 `thin_film.ior` 参数控制。

5. **毛发模型**: 支持 Chiang (2016) 和 Huang (2022) 两种物理毛发 BSDF 模型，包括黑色素浓度参数化和随机化。

6. **体积相函数**: Mie 散射通过拟合参数分解为 Henyey-Greenstein 和 Draine 的混合。

## 关联文件

- `kernel/svm/svm.h` — SVM 主调度器，调用本文件中的闭包函数
- `kernel/closure/bsdf.h` — BSDF 各类型的具体实现
- `kernel/closure/bssrdf.h` — BSSRDF 次表面散射实现
- `kernel/closure/volume.h` — 体积相函数实现
