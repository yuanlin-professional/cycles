# bsdf_oren_nayar.h - Oren-Nayar 漫反射双向散射分布函数（BSDF）实现

## 概述

本文件实现了改进版 Oren-Nayar 漫反射模型，用于模拟粗糙表面的漫反射散射行为。实现基于 Fujii Yasuhiro 的改进 Oren-Nayar 模型，并附加了基于 OpenPBR 规范的多重散射能量补偿项，确保粗糙漫反射表面不会出现能量损失。相比 Lambert 模型，Oren-Nayar 能更准确地还原粗糙表面在掠射角下的明暗变化。

## 类与结构体

### `struct OrenNayarBsdf`
Oren-Nayar BSDF 数据结构，继承自 `SHADER_CLOSURE_BASE`：
- `roughness` — 表面粗糙度参数（0 到 1），0 时退化为 Lambert 模型
- `a` — 预计算系数，`a = 1 / (pi + sigma * (pi/2 - 2/3))`
- `b` — 预计算系数，`b = sigma * a`，其中 `sigma` 为钳位后的粗糙度
- `multiscatter_term` — 多重散射能量补偿项（光谱值），基于反照率和能量损失计算

## 核心函数

### `bsdf_oren_nayar_G(cosTheta)`
辅助几何函数，计算给定余弦角度下的 G 项。在 `cosTheta` 极小时（< 1e-6）使用泰勒展开避免 `tan(theta)` 的数值问题。

### `bsdf_oren_nayar_get_intensity(sc, n, v, l)`
核心强度计算函数。计算给定法线 `n`、视线方向 `v` 和光线方向 `l` 下的散射强度：
1. 计算单次散射项：`a + b * t`，其中 `t` 为与视线和光线方向相关的角度项
2. 计算多重散射项：`multiscatter_term * (1 - El)`，其中 `El` 为光线方向的能量
3. 最终结果为 `nl * (single_scatter + multi_scatter)`

### `bsdf_oren_nayar_setup(sd, bsdf, color)`
闭包初始化函数：
1. 将粗糙度钳位到 [0, 1]
2. 预计算系数 `a` 和 `b`
3. 根据反照率（`color`）计算多重散射能量补偿项 `Ems`，基于平均能量 `Eavg` 和视线方向能量 `Ev`
4. 返回 `SD_BSDF | SD_BSDF_HAS_EVAL` 标志

### `bsdf_oren_nayar_eval(sc, wi, wo, pdf)`
BSDF 求值函数。当出射方向在法线正半球时，PDF 为余弦分布 `cos(N,O) / pi`，返回 `bsdf_oren_nayar_get_intensity` 的结果。

### `bsdf_oren_nayar_sample(sc, Ng, wi, rand, eval, wo, pdf)`
BSDF 采样函数。使用余弦加权半球采样（`sample_cos_hemisphere`）生成出射方向，检查几何法线可见性后返回强度。标签为 `LABEL_REFLECT | LABEL_DIFFUSE`。

## 依赖关系
- **内部头文件**:
  - `kernel/types.h` — 内核类型定义
  - `kernel/sample/mapping.h` — 采样映射（余弦半球采样）
- **被引用**:
  - `kernel/closure/bsdf.h` — 闭包统一调度入口

## 实现细节 / 关键算法

### 改进 Oren-Nayar 模型
实现基于 Fujii Yasuhiro 的改进版本（https://mimosa-pudica.net/improved-oren-nayar.html），相比原始 Oren-Nayar (1994) 模型更简洁高效。核心公式简化为：
```
f = a + b * t
```
其中 `t` 由入射和出射方向的几何关系决定。当 `t > 0` 时除以 `max(nl, nv)` 进行归一化。

### 多重散射能量补偿
基于 OpenPBR 规范（https://academysoftwarefoundation.github.io/OpenPBR），通过以下步骤补偿单次散射模型的能量损失：
1. 计算反照率饱和值 `albedo = saturate(color)`
2. 计算平均能量 `Eavg = a * pi + ((2pi - 5.6) / 3) * b`
3. 计算多重散射能量 `Ems = (1/pi) * albedo^2 * (Eavg / (1 - Eavg)) / (1 - albedo * (1 - Eavg))`
4. 在 setup 时根据视线方向预乘 `(1 - Ev)`，在 eval 时根据光线方向乘以 `(1 - El)`

### 采样策略
采用简单的余弦加权半球采样，与 Lambert 模型共享采样策略。虽然不是精确的重要性采样，但对于漫反射闭包而言足够高效。

## 关联文件
- `kernel/closure/bsdf.h` — 闭包调度入口，根据 `CLOSURE_BSDF_OREN_NAYAR_ID` 调用本文件函数
- `kernel/sample/mapping.h` — 提供 `sample_cos_hemisphere` 函数
