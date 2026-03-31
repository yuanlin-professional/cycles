# volume_henyey_greenstein.h - Henyey-Greenstein 体积相位函数闭包

## 概述

本文件实现了 Cycles 渲染器中的 Henyey-Greenstein（HG）体积相位函数闭包。HG 相位函数是体积渲染中最常用的参数化散射模型，通过单一各向异性参数 g 控制前向/后向散射偏好。它被广泛应用于烟雾、云层、皮肤次表面散射等各类体积效果的模拟。

## 类与结构体

### `HenyeyGreensteinVolume`
Henyey-Greenstein 体积闭包的数据结构，继承自 `SHADER_CLOSURE_VOLUME_BASE`。

| 成员 | 类型 | 说明 |
|------|------|------|
| `g` | `float` | 各向异性参数。g=0 为各向同性散射，g>0 前向散射，g<0 后向散射 |

编译时通过 `static_assert` 确保 `HenyeyGreensteinVolume` 不超过 `ShaderVolumeClosure` 的大小。

## 核心函数

### `volume_henyey_greenstein_setup`
```cpp
ccl_device int volume_henyey_greenstein_setup(ccl_private HenyeyGreensteinVolume *volume)
```
初始化 HG 闭包：
1. 设置类型为 `CLOSURE_VOLUME_HENYEY_GREENSTEIN_ID`
2. 将 g 钳制到 `(-1+1e-3, 1-1e-3)` 范围，保留符号，避免 delta 函数奇异值
3. 返回 `SD_SCATTER` 标志

### `volume_henyey_greenstein_eval`
```cpp
ccl_device Spectrum volume_henyey_greenstein_eval(const ccl_private ShaderData *sd,
                                                  const ccl_private ShaderVolumeClosure *svc,
                                                  const float3 wo, ccl_private float *pdf)
```
在给定出射方向上求值。计算入射方向 `-sd->wi` 与出射方向 `wo` 的夹角余弦，调用 `phase_henyey_greenstein` 获取 PDF。

### `volume_henyey_greenstein_sample`
```cpp
ccl_device int volume_henyey_greenstein_sample(const ccl_private ShaderData *sd,
                                               const ccl_private ShaderVolumeClosure *svc,
                                               const float2 rand,
                                               ccl_private Spectrum *eval,
                                               ccl_private float3 *wo, ccl_private float *pdf)
```
重要性采样。调用 `phase_henyey_greenstein_sample` 生成出射方向。完美重要性采样（`eval = pdf`），不引入额外方差。返回 `LABEL_VOLUME_SCATTER`。

## 依赖关系
- **内部头文件**:
  - `kernel/types.h` — 内核类型定义
  - `kernel/closure/volume_util.h` — 体积工具函数（`phase_henyey_greenstein` 等）
- **被引用**:
  - `src/kernel/closure/volume.h` — 体积闭包调度中心

## 实现细节 / 关键算法

### Henyey-Greenstein 相位函数
经典 HG 相位函数的数学表达式：

```
p(cos_theta) = (1 - g^2) / (4pi * (1 + g^2 - 2g*cos_theta)^(3/2))
```

关键性质：
- 归一化条件：在全球面上积分等于 1
- g = 0 时退化为各向同性散射 `1/(4pi)`
- g -> 1 时趋近于 delta 函数（完美前向散射）
- g -> -1 时趋近于 delta 函数（完美后向散射）

### 解析重要性采样
HG 函数的 CDF 有解析反函数，采样公式为：
```
k = (1 - g^2) / (1 - g + 2g * rand)
cos_theta = (1 + g^2 - k^2) / (2g)
```
当 `|g| < 1e-3` 时简化为均匀球面采样 `cos_theta = 1 - 2*rand`。

### g 参数钳制
g 被钳制到 `(-1+1e-3, 1-1e-3)` 范围以避免两个问题：
1. `|g| = 1` 导致 delta 函数，使 PDF 在某些方向上趋于无穷
2. 采样公式中出现除零（分母包含 `2g`）

## 关联文件
- `src/kernel/closure/volume_util.h` — HG 相位函数和采样的底层数学实现
- `src/kernel/closure/volume.h` — 体积闭包调度中心
- `src/kernel/closure/volume_draine.h` — Draine 相位函数（HG 的推广形式）
- `src/kernel/closure/volume_rayleigh.h` — Rayleigh 散射（g=0 的物理模型）
