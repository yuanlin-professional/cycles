# volume_rayleigh.h - Rayleigh 体积相位函数闭包

## 概述

本文件实现了 Cycles 渲染器中的 Rayleigh 体积相位函数闭包。Rayleigh 散射描述了光与远小于其波长的粒子相互作用时的散射行为，是大气散射的基本模型之一。该相位函数无参数，其特征为前后方向等强度散射，侧面（90 度）散射最弱，适用于模拟晴空大气散射等效果。

## 类与结构体

### `RayleighVolume`
Rayleigh 体积闭包的数据结构，继承自 `SHADER_CLOSURE_VOLUME_BASE`。

该结构体没有额外成员——Rayleigh 散射是一个固定的无参数相位函数。

编译时通过 `static_assert` 确保 `RayleighVolume` 不超过 `ShaderVolumeClosure` 的大小。

## 核心函数

### `volume_rayleigh_setup`
```cpp
ccl_device int volume_rayleigh_setup(ccl_private RayleighVolume *volume)
```
初始化 Rayleigh 闭包。设置类型为 `CLOSURE_VOLUME_RAYLEIGH_ID`，返回 `SD_SCATTER` 标志。无需参数钳制。

### `volume_rayleigh_eval`
```cpp
ccl_device Spectrum volume_rayleigh_eval(const ccl_private ShaderData *sd,
                                         const float3 wo, ccl_private float *pdf)
```
在给定出射方向上求值。计算入射方向 `-sd->wi` 与出射方向 `wo` 的夹角余弦，调用 `phase_rayleigh` 获取 PDF。注意该函数不需要闭包指针参数（无参数）。

### `volume_rayleigh_sample`
```cpp
ccl_device int volume_rayleigh_sample(const ccl_private ShaderData *sd,
                                      const float2 rand,
                                      ccl_private Spectrum *eval,
                                      ccl_private float3 *wo, ccl_private float *pdf)
```
重要性采样。调用 `phase_rayleigh_sample` 生成出射方向。完美重要性采样（`eval = pdf`）。返回 `LABEL_VOLUME_SCATTER`。

## 依赖关系
- **内部头文件**:
  - `kernel/types.h` — 内核类型定义
  - `kernel/closure/volume_util.h` — 体积工具函数（`phase_rayleigh` 等）
- **被引用**:
  - `src/kernel/closure/volume.h` — 体积闭包调度中心

## 实现细节 / 关键算法

### Rayleigh 相位函数
数学表达式（参见 https://doi.org/10.1364/JOSAA.28.002436 ）：

```
p(cos_theta) = (3 / 16pi) * (1 + cos^2(theta))
```

关键性质：
- 完全对称：`p(theta) = p(pi - theta)`，前向和后向散射强度相等
- 在 theta = 0 和 theta = pi 处最强（归一化后为 `3/(8pi)`）
- 在 theta = pi/2 处最弱（`3/(16pi)`）
- 无参数，形状固定

### 解析重要性采样
采样使用立方根反函数法。关键技巧是利用快速逆立方根（Quake 风格的 `fast_inv_cbrtf`）替代标准 `cbrtf`：
```
a = 2 - 4 * rand
inv_u = -fast_inv_cbrtf(sqrt(1 + a^2) + a)
cos_theta = 1/inv_u - inv_u
```

这种方法避免了 Metal（Apple GPU）上缺少 `cbrtf` 的问题，同时保持了数值精度。

## 关联文件
- `src/kernel/closure/volume_util.h` — Rayleigh 相位函数和采样的底层数学实现
- `src/kernel/closure/volume.h` — 体积闭包调度中心
- `src/kernel/closure/volume_draine.h` — Draine 相位函数（g=0, alpha=1 时退化为 Rayleigh）
- `src/kernel/closure/volume_henyey_greenstein.h` — HG 相位函数（g=0 时为各向同性，非 Rayleigh）
