# volume_fournier_forand.h - Fournier-Forand 体积相位函数闭包

## 概述

本文件实现了 Cycles 渲染器中的 Fournier-Forand 体积相位函数闭包。Fournier-Forand 相位函数专为模拟水体中悬浮粒子的光散射而设计，通过粒子折射率（IOR）和后向散射分数（B）两个物理参数控制散射分布。该模型广泛应用于海洋光学和水下渲染。

## 类与结构体

### `FournierForandVolume`
Fournier-Forand 体积闭包的数据结构，继承自 `SHADER_CLOSURE_VOLUME_BASE`。

| 成员 | 类型 | 说明 |
|------|------|------|
| `c1` | `float` | 预计算系数（粒子折射率 n） |
| `c2` | `float` | 预计算系数（v 参数） |
| `c3` | `float` | 预计算系数（归一化修正项） |

编译时通过 `static_assert` 确保 `FournierForandVolume` 不超过 `ShaderVolumeClosure` 的大小。

## 核心函数

### `volume_fournier_forand_setup`
```cpp
ccl_device int volume_fournier_forand_setup(ccl_private FournierForandVolume *volume,
                                            float B, float IOR)
```
初始化 Fournier-Forand 闭包：
1. 设置类型为 `CLOSURE_VOLUME_FOURNIER_FORAND_ID`
2. 钳制后向散射分数 B 到 `[0, 0.5-1e-3]`，避免 delta 函数
3. 钳制折射率 IOR 到 `[1+1e-3, ...]`
4. 调用 `phase_fournier_forand_coeffs` 预计算三个系数 `(c1, c2, c3)`
5. 返回 `SD_SCATTER` 标志

### `volume_fournier_forand_eval`
```cpp
ccl_device Spectrum volume_fournier_forand_eval(const ccl_private ShaderData *sd,
                                                const ccl_private ShaderVolumeClosure *svc,
                                                const float3 wo, ccl_private float *pdf)
```
在给定出射方向上求值。从闭包中重建系数向量，调用 `phase_fournier_forand` 计算 PDF。

### `volume_fournier_forand_sample`
```cpp
ccl_device int volume_fournier_forand_sample(const ccl_private ShaderData *sd,
                                             const ccl_private ShaderVolumeClosure *svc,
                                             const float2 rand,
                                             ccl_private Spectrum *eval,
                                             ccl_private float3 *wo, ccl_private float *pdf)
```
重要性采样。调用 `phase_fournier_forand_sample`（内部使用 Newton-Raphson 法求解 CDF 反函数）。完美重要性采样，`eval = pdf`。返回 `LABEL_VOLUME_SCATTER`。

## 依赖关系
- **内部头文件**:
  - `kernel/types.h` — 内核类型定义
  - `kernel/closure/volume_util.h` — 体积工具函数（`phase_fournier_forand` 等）
- **被引用**:
  - `src/kernel/closure/volume.h` — 体积闭包调度中心

## 实现细节 / 关键算法

### 预计算系数
在 `setup` 阶段预计算三个系数而非存储原始参数 `(B, IOR)`，这是因为：
- 系数 `c1`（= IOR）在求值时用于计算 delta 函数
- 系数 `c2`（= v 参数）控制散射分布形状
- 系数 `c3`（归一化修正项）用于 Rayleigh 修正项

预计算避免了每次求值时的重复对数和幂运算。

### Fournier-Forand 相位函数
基于 https://doi.org/10.1117/12.366488 的模型，核心参数：
- 粒子折射率 n 控制前向/后向散射的相对强度
- 后向散射分数 B = b_b/b 控制后向散射占比
- v 参数通过 B 和 delta(90 度) 求解

### Newton-Raphson 采样
由于 FF 相位函数的 CDF 无解析反函数，采样使用 Newton-Raphson 迭代法（最多 20 次迭代）。初始猜测为 50 度角（cos_theta = 0.643），对于前向散射主导的情况能较快收敛。

## 关联文件
- `src/kernel/closure/volume_util.h` — 包含底层相位函数、系数计算和采样实现
- `src/kernel/closure/volume.h` — 体积闭包调度中心
- `src/kernel/closure/volume_henyey_greenstein.h` — HG 相位函数（更简单的替代模型）
