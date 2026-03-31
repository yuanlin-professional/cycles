# fresnel.h - 菲涅尔与层权重节点的SVM实现

## 概述

`fresnel.h` 实现了着色器虚拟机(SVM)中的菲涅尔(Fresnel)节点和层权重(Layer Weight)节点。这两个节点用于根据观察角度计算表面的反射/折射混合因子，是实现真实感材质的重要输入节点。它们常被用于控制闭包混合或作为遮罩输入。

## 核心函数

### `svm_node_fresnel`
- **签名**: `ccl_device_noinline void svm_node_fresnel(ShaderData *sd, float *stack, const uint ior_offset, const uint ior_value, const uint node)`
- **功能**: 计算电介质菲涅尔反射系数。
- **参数**:
  - `ior_offset` / `ior_value` — 折射率(IOR)，从栈或常量获取
  - `normal_offset` — 可选自定义法线，默认使用 `sd->N`
- **算法**:
  1. 读取折射率 `eta`，钳制最小值为 `1e-5`
  2. 如果是背面(`SD_BACKFACING`)，取 `1/eta`
  3. 调用 `fresnel_dielectric_cos(dot(sd->wi, N), eta)` 计算菲涅尔系数
  4. 输出单个浮点值（反射因子，范围 0~1）

### `svm_node_layer_weight`
- **签名**: `ccl_device_noinline void svm_node_layer_weight(ShaderData *sd, float *stack, const uint4 node)`
- **功能**: 层权重节点，提供两种模式计算视角相关的混合因子。
- **模式**:
  - **Fresnel 模式** (`NODE_LAYER_WEIGHT_FRESNEL`): 将 `blend` 参数映射为折射率，计算菲涅尔系数。`eta = max(1-blend, 1e-5)`，背面时取 `eta` 本身，正面时取 `1/eta`。
  - **Facing 模式** (默认): 计算 `1 - |dot(wi, N)|^blend` 的效果。`blend` 在 0~1 范围内进行非线性重映射：小于 0.5 时 `blend *= 2`，大于 0.5 时 `blend = 0.5/(1-blend)`，使得 `blend=0.5` 为线性转折点。

## 依赖关系

- **内部头文件**:
  - `kernel/closure/bsdf_util.h` — 提供 `fresnel_dielectric_cos()` 函数
  - `kernel/svm/util.h` — SVM 栈操作工具
- **被引用**: `kernel/svm/svm.h`

## 实现细节 / 关键算法

1. **菲涅尔系数计算**: 使用 `fresnel_dielectric_cos()` 函数，该函数基于 Fresnel 方程的精确解（非 Schlick 近似），接受入射角余弦和折射率作为参数。

2. **背面处理**: 两个函数都会检测 `SD_BACKFACING` 标志。对于菲涅尔节点，背面时将折射率取倒数（即从介质内部观察）。对于层权重的 Fresnel 模式，正面和背面使用不同的 eta 映射。

3. **Layer Weight Facing 模式的非线性映射**: `blend` 参数在 0.5 处具有不同的映射行为：
   - `blend < 0.5`: `blend = 2 * blend`（线性缩放）
   - `blend >= 0.5`: `blend = 0.5 / (1 - blend)`（趋于无穷大的映射）

   这使得小的 blend 值产生柔和的边缘效果，而接近 1.0 的值产生非常尖锐的边缘效果。

## 关联文件

- `kernel/closure/bsdf_util.h` — 底层菲涅尔计算实现
- `kernel/svm/svm.h` — SVM 主调度器
