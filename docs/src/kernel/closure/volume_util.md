# volume_util.h - 体积相位函数底层数学工具库

## 概述

本文件是 Cycles 渲染器中所有体积相位函数的底层数学实现库。它包含了 Henyey-Greenstein、Rayleigh、Draine 和 Fournier-Forand 四种相位函数的求值与采样算法，以及通用的方向采样辅助函数和近似 Mie 散射的参数拟合函数。所有体积闭包头文件均依赖本文件。

## 类与结构体

本文件不定义结构体。

## 核心函数

### 通用辅助函数

#### `phase_sample_direction`
```cpp
ccl_device float3 phase_sample_direction(const float3 D, float cos_theta, float rand)
```
给定参考方向 D、散射角余弦 cos_theta 和随机数 rand，通过球坐标到笛卡尔坐标的转换生成采样方向。构建以 D 为轴的正交坐标系，在该坐标系下将 `(cos_theta, phi)` 映射为三维方向。

### Henyey-Greenstein 相位函数

#### `phase_henyey_greenstein`
```cpp
ccl_device float phase_henyey_greenstein(float cos_theta, float g)
```
HG 相位函数求值：`(1-g^2) / (4pi * (1+g^2-2g*cos_theta)^(3/2))`。当 `|g| < 1e-3` 时简化为各向同性散射 `1/(4pi)`。

#### `phase_henyey_greenstein_sample`
HG 的解析重要性采样。利用 CDF 解析反函数计算采样的 cos_theta。

### Rayleigh 相位函数

#### `phase_rayleigh`
```cpp
ccl_device float phase_rayleigh(float cos_theta)
```
Rayleigh 相位函数求值：`(3/16pi) * (1 + cos^2(theta))`。

#### `phase_rayleigh_sample`
使用快速逆立方根（`fast_inv_cbrtf`）实现的解析采样。利用 `u - 1/u` 的代数恒等式避免直接调用 `cbrtf`。

### Draine 相位函数

#### `phase_draine`
```cpp
ccl_device float phase_draine(float cos_theta, float g, float alpha)
```
Draine 相位函数求值，为 HG 的推广。包含特殊情况处理：
- `|g| < 1e-3, alpha > 0.999` -> 退化为 Rayleigh
- `|alpha| < 1e-3` -> 退化为 HG

#### `phase_draine_sample_cos`
解析采样的 cos_theta 计算，改编自 NVIDIA 近似 Mie 散射论文的 HLSL 代码。对 `|g| < 1e-2` 使用简化的立方根方法。

#### `phase_draine_sample`
完整的 Draine 采样，包含特殊情况的退化路径。

### Fournier-Forand 相位函数

#### `phase_fournier_forand_delta`
计算 FF 模型中的 delta 参数：`u / (3 * (n-1)^2)`，其中 `u = 4 * sin^2(theta/2)`。

#### `phase_fournier_forand_coeffs`
从物理参数 `(B, IOR)` 预计算三个系数 `(n, v, pf_coeff)`。

#### `phase_fournier_forand_impl`
FF 相位函数的核心实现，处理 delta 接近 1 时的 Taylor 展开特殊情况。

#### `phase_fournier_forand`
FF 相位函数求值入口，处理 `|cos_theta| >= 1` 边界情况后调用 `_impl`。

#### `phase_fournier_forand_newton`
Newton-Raphson 法求解 FF 的 CDF 反函数（最多 20 次迭代），用于重要性采样。

#### `phase_fournier_forand_sample`
FF 的完整采样函数。

### 近似 Mie 散射

#### `phase_mie_fitted_parameters`
```cpp
ccl_device void phase_mie_fitted_parameters(float d,
    ccl_private float *g_HG, ccl_private float *g_D,
    ccl_private float *alpha, ccl_private float *w)
```
基于 NVIDIA 论文"An Approximate Mie Scattering Function for Fog and Cloud Rendering"的参数拟合。将水滴直径 d（0-50 微米）映射到 Draine + Henyey-Greenstein 混合模型的四个参数。分四个区间（d <= 0.1, 0.1 < d < 1.5, 1.5 <= d < 5.0, d >= 5.0）使用不同的拟合公式。

## 依赖关系
- **内部头文件**:
  - `util/math_fast.h` — 快速数学函数（`fast_inv_cbrtf`、`fast_sinf`、`fast_cosf` 等）
  - `util/projection.h` — 投影与方向转换工具（`spherical_cos_to_direction` 等）
- **被引用**:
  - `src/kernel/closure/volume_draine.h` — Draine 体积闭包
  - `src/kernel/closure/volume_fournier_forand.h` — Fournier-Forand 体积闭包
  - `src/kernel/closure/volume_henyey_greenstein.h` — Henyey-Greenstein 体积闭包
  - `src/kernel/closure/volume_rayleigh.h` — Rayleigh 体积闭包

## 实现细节 / 关键算法

### Fournier-Forand 的 Newton-Raphson 采样
FF 相位函数的 CDF 无解析反函数，因此采用数值方法：
1. 初始猜测 cos_theta = 0.643（50 度角）
2. 每次迭代同时计算 CDF 和 PDF
3. 更新公式：`cos_theta_new = cos_theta + (cdf - rand) / (2pi * pdf)`
4. 当 cos_theta 趋近 1.0 时使用缓慢逼近策略避免越界
5. 最多 20 次迭代，收敛容差 1e-6

### 快速数学函数的使用
大量使用 `fast_*` 系列函数（如 `fast_inv_cbrtf`、`fast_sinf`、`fast_cosf`、`fast_logf`、`fast_expf`），在保持可接受精度的前提下显著提升 GPU 计算性能。特别是 `fast_inv_cbrtf` 使用了类似 Quake III 快速逆平方根的位操作技巧。

### Mie 散射参数拟合
针对不同水滴直径范围使用分段拟合，确保参数在各区间内平滑变化：
- d <= 0.1：多项式和三角函数拟合
- 0.1 < d < 1.5：对数域上的三角函数拟合
- 1.5 <= d < 5.0：对数域上的有理函数和三角函数拟合
- d >= 5.0：指数拟合（渐近行为）

### Taylor 展开处理数值稳定性
FF 相位函数在 delta 接近 1 时分母趋近零，使用一阶 Taylor 展开避免数值不稳定。类似地，HG 和 Draine 在 g 接近零时切换到简化公式。

## 关联文件
- `src/kernel/closure/volume_henyey_greenstein.h` — 使用 `phase_henyey_greenstein` 和 `phase_henyey_greenstein_sample`
- `src/kernel/closure/volume_rayleigh.h` — 使用 `phase_rayleigh` 和 `phase_rayleigh_sample`
- `src/kernel/closure/volume_draine.h` — 使用 `phase_draine` 和 `phase_draine_sample`
- `src/kernel/closure/volume_fournier_forand.h` — 使用 `phase_fournier_forand` 和 `phase_fournier_forand_sample`
- `util/math_fast.h` — 快速数学实现
