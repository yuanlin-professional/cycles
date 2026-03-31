# bsdf_principled_hair_chiang.h - Chiang 毛发散射模型实现

## 概述

本文件实现了基于 Chiang et al. (2015) 论文 "A practical and controllable hair and fur model for production path tracing" 的毛发双向散射分布函数（BSDF）。该模型将毛发散射分解为四个分量：主反射（R）、透射（TT）、二次反射（TRT）和残余项（TRRT+），通过纵向散射函数 Mp 和方位角散射函数 Np 的乘积来描述各分量的散射行为。此模型是 Cycles 中 Principled Hair BSDF 的经典实现方案。

## 类与结构体

### `struct ChiangHairBSDF`
Chiang 毛发 BSDF 数据结构，继承自 `SHADER_CLOSURE_BASE`：
- `sigma` — 吸收系数（光谱值），控制毛发颜色
- `v` — 底层 Logistic 分布的方差，控制纵向粗糙度
- `s` — 底层 Logistic 分布的尺度因子，控制方位角粗糙度
- `alpha` — 角质层倾斜角度（cuticle tilt），影响各散射分量的纵向偏移
- `eta` — 折射率（IOR），通常约 1.55
- `m0_roughness` — 主反射（R）分量的有效方差，可独立于整体粗糙度调节
- `h` — 方位角偏移量，表示光线在毛发截面上的入射位置（-1 到 1）

## 核心函数

### 工具函数
- **`delta_phi(p, gamma_o, gamma_t)`** — 计算第 p 阶散射在法线平面内的方向变化量。
- **`wrap_angle(a)`** — 将角度重映射到 [-pi, pi] 范围。

### 概率分布函数
- **`logistic(x, s)`** — Logistic 分布函数。
- **`logistic_cdf(x, s)`** — Logistic 累积分布函数，带溢出保护。
- **`trimmed_logistic(x, s)`** — 截断到 [-pi, pi] 的 Logistic 分布。
- **`sample_trimmed_logistic(u, s)`** — 截断 Logistic 分布的逆 CDF 采样。

### Bessel 函数
- **`bessel_I0(x)`** — 第一类修正 Bessel 函数 I0 的数值近似（幂级数展开至第 10 阶）。
- **`log_bessel_I0(x)`** — I0 的对数值，当 `x > 12` 时使用渐近展开避免溢出。

### 散射函数
- **`azimuthal_scattering(phi, p, s, gamma_o, gamma_t)`** — 方位角散射函数 Np，基于截断 Logistic 分布。
- **`longitudinal_scattering(sin_theta_i, cos_theta_i, sin_theta_o, cos_theta_o, v)`** — 纵向散射函数 Mp，基于修正 Bessel 函数。当方差 `v <= 0.1` 时使用对数空间计算避免数值问题。

### 衰减与角度计算
- **`hair_attenuation(kg, f, T, Ap, Ap_energy)`** — 根据菲涅尔项 `f` 和透射率 `T`，计算各阶散射（R、TT、TRT、TRRT+）的衰减系数 `Ap` 和对应的采样权重 `Ap_energy`。
- **`hair_alpha_angles(sin_theta_o, cos_theta_o, alpha, angles)`** — 根据角质层倾斜角 `alpha` 计算各阶散射的倾斜正弦和余弦值。

### 闭包接口
- **`bsdf_hair_chiang_setup(sd, bsdf)`** — 闭包初始化：钳位参数、将粗糙度映射为方差和尺度因子、计算局部坐标系和 `h` 值。
- **`bsdf_hair_chiang_eval(kg, sd, sc, wo, pdf)`** — BSDF 求值：分别计算 R、TT、TRT 三个分量（使用各自的方差和角质层偏移）以及 TRRT+ 残余项。
- **`bsdf_hair_chiang_sample(kg, sc, sd, rand, eval, wo, pdf, sampled_roughness)`** — BSDF 采样：按衰减能量权重选择散射分量，然后在纵向和方位角上分别采样。
- **`bsdf_hair_chiang_blur(sc, roughness)`** — 实现 Filter Glossy，将粗糙度参数提升到给定最小值。
- **`bsdf_hair_chiang_albedo(sd, sc)`** — 计算毛发反照率近似值。

## 依赖关系
- **内部头文件**:
  - `kernel/types.h` — 内核类型定义
  - `kernel/closure/bsdf_util.h` — BSDF 工具函数（菲涅尔公式等）
  - `kernel/util/colorspace.h` — 颜色空间转换
- **被引用**:
  - `kernel/closure/bsdf.h` — 闭包统一调度入口
  - `kernel/closure/bsdf_principled_hair_huang.h` — Huang 毛发模型复用纵向散射函数

## 实现细节 / 关键算法

### 条件编译
`bsdf_hair_chiang_setup` 被 `#ifdef __HAIR__` 包裹，仅在启用毛发渲染功能时编译。其余函数（eval、sample 等）不受此限制。

### 粗糙度到方差的映射
在 setup 阶段，用户空间的粗糙度参数通过多项式映射转换为底层分布参数：
```
v = (0.726 * v + 0.812 * v^2 + 3.700 * v^20)^2
s = (0.265 * s + 1.194 * s^2 + 5.372 * s^22) * sqrt(pi/8)
```
这些非线性映射确保了粗糙度参数在用户层面的线性感知变化。

### 纵向散射函数 Mp
基于 von Mises-Fisher 分布的修改版，使用修正 Bessel 函数 I0。当方差极小（`v <= 0.1`）时切换到对数空间计算以避免数值溢出。

### 方位角散射函数 Np
使用截断到 [-pi, pi] 的 Logistic 分布来近似各阶散射的方位角分布。方位角偏移量由几何光学的折射路径决定。

### 散射分量衰减
基于菲涅尔反射和 Beer 定律吸收来计算各阶散射的能量衰减：
- **R**: 直接菲涅尔反射，`Ap[0] = f`
- **TT**: 双重透射加吸收，`Ap[1] = (1-f)^2 * T`
- **TRT**: 透射-反射-透射，`Ap[2] = (1-f)^2 * T^2 * f`
- **TRRT+**: 残余项作为几何级数求和

### 局部坐标系
以曲线切线方向 `X = normalize(dPdu)` 为主轴，构建正交坐标系。`h` 值表示光线相对于毛发截面中心的偏移，对于 Ribbon 类型曲线使用 `-sd->v`，对于实际曲线几何使用 `dot(cross(Ng, X), Z)`。

## 关联文件
- `kernel/closure/bsdf_util.h` — 提供 `fresnel_dielectric_cos` 等菲涅尔计算函数
- `kernel/closure/bsdf.h` — 闭包调度入口
- `kernel/closure/bsdf_principled_hair_huang.h` — Huang 模型引用本文件并复用 `longitudinal_scattering` 函数
