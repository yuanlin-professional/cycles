# volume_draine.h - Draine 体积相位函数闭包

## 概述

本文件实现了 Cycles 渲染器中的 Draine 体积相位函数闭包。Draine 相位函数是 Henyey-Greenstein 函数的推广形式，通过额外的 alpha 参数引入余弦平方项，可以在 Henyey-Greenstein 散射（alpha=0）和 Rayleigh 散射（g=0, alpha=1）之间灵活插值。它特别适用于模拟星际尘埃等粒子的散射行为。

## 类与结构体

### `DraineVolume`
Draine 体积闭包的数据结构，继承自 `SHADER_CLOSURE_VOLUME_BASE`。

| 成员 | 类型 | 说明 |
|------|------|------|
| `g` | `float` | 各向异性参数，控制散射方向偏好（正值前向散射，负值后向散射） |
| `alpha` | `float` | 二阶矩参数，控制散射分布形状 |

编译时通过 `static_assert` 确保 `DraineVolume` 不超过 `ShaderVolumeClosure` 的大小。

## 核心函数

### `volume_draine_setup`
```cpp
ccl_device int volume_draine_setup(ccl_private DraineVolume *volume)
```
初始化 Draine 体积闭包：
1. 设置类型为 `CLOSURE_VOLUME_DRAINE_ID`
2. 将各向异性参数 g 钳制到 `(-1+1e-3, 1-1e-3)` 范围，保留符号
3. 返回 `SD_SCATTER` 标志

### `volume_draine_eval`
```cpp
ccl_device Spectrum volume_draine_eval(const ccl_private ShaderData *sd,
                                       const ccl_private ShaderVolumeClosure *svc,
                                       const float3 wo, ccl_private float *pdf)
```
在给定出射方向上求值 Draine 相位函数。计算入射方向（`-sd->wi`，因为 `wi` 指向观察者）与出射方向的夹角余弦，调用 `phase_draine` 获取 PDF 值。

### `volume_draine_sample`
```cpp
ccl_device int volume_draine_sample(const ccl_private ShaderData *sd,
                                    const ccl_private ShaderVolumeClosure *svc,
                                    const float2 rand,
                                    ccl_private Spectrum *eval,
                                    ccl_private float3 *wo, ccl_private float *pdf)
```
重要性采样。调用 `phase_draine_sample` 生成出射方向。由于是完美重要性采样，`eval` 等于 `pdf`。返回 `LABEL_VOLUME_SCATTER`。

## 依赖关系
- **内部头文件**:
  - `kernel/types.h` — 内核类型定义
  - `kernel/closure/volume_util.h` — 体积工具函数（`phase_draine`、`phase_draine_sample` 等）
- **被引用**:
  - `src/kernel/closure/volume.h` — 体积闭包调度中心

## 实现细节 / 关键算法

### Draine 相位函数
Draine 相位函数（参见 https://doi.org/10.1086/379118 ）的数学表达式为：

```
p(cos_theta) = (1 - g^2)(1 + alpha * cos^2(theta))
               / ((1 + (alpha * (1 + 2g^2)) / 3) * 4pi * (1 + g^2 - 2g*cos_theta)^(3/2))
```

- 当 `alpha = 0` 时退化为标准 Henyey-Greenstein 函数
- 当 `g = 0, alpha = 1` 时退化为 Rayleigh 散射
- 当 `alpha = 1` 时为 Cornette-Shanks 函数

### 完美重要性采样
采样方法基于 NVIDIA 的近似 Mie 散射论文中提供的 HLSL 代码，通过解析求解 CDF 的反函数实现完美重要性采样（eval = pdf），不会引入额外方差。

### 方向约定
注意 `sd->wi` 指向观察者（即光的传播反方向），因此在计算散射角余弦时使用 `-sd->wi`。

## 关联文件
- `src/kernel/closure/volume_util.h` — 包含 `phase_draine` 和 `phase_draine_sample` 的底层实现
- `src/kernel/closure/volume.h` — 体积闭包调度中心，通过 switch-case 调用本文件函数
- `src/kernel/closure/volume_henyey_greenstein.h` — HG 相位函数（Draine 的特殊情况）
- `src/kernel/closure/volume_rayleigh.h` — Rayleigh 相位函数（Draine 的另一特殊情况）
