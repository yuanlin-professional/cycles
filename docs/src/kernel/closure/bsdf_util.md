# bsdf_util.h - 双向散射分布函数(BSDF)通用工具函数库

## 概述

本文件是 Cycles 渲染器中 BSDF 闭包系统的核心工具库，改编自 Open Shading Language。它提供了菲涅尔反射率计算、薄膜虹彩效果、高光法线修正、毛发着色参数以及闭包分层等大量通用函数。几乎所有 BSDF 闭包实现都直接或间接依赖本文件。

## 类与结构体

### `FresnelThinFilm`
薄膜干涉参数结构体。

| 成员 | 类型 | 说明 |
|------|------|------|
| `thickness` | `float` | 薄膜厚度（纳米） |
| `ior` | `float` | 薄膜折射率 |

### `complex<T>`
模板复数类型，用于导体菲涅尔和虹彩计算中的相位运算。

| 成员 | 类型 | 说明 |
|------|------|------|
| `re` | `T` | 实部 |
| `im` | `T` | 虚部 |

支持运算符 `*=`（复数乘法）和 `*`（标量乘法）。

### `fresnel_info<T>`
模板元编程辅助结构，用于在编译期区分导体（`Spectrum`）和电介质（`float`）菲涅尔计算路径。

## 核心函数

### 电介质菲涅尔

#### `fresnel_dielectric_polarized`
计算偏振光的菲涅尔反射率（S 偏振和 P 偏振）。使用 Snell 定律计算折射角，并可选返回反射相移的余弦值。处理全内反射（TIR）情况。

#### `fresnel_dielectric`
非偏振光的菲涅尔反射率，为 S/P 偏振的平均值。

#### `fresnel_dielectric_cos`
不显式计算折射方向的简化菲涅尔计算。

#### `fresnel_dielectric_Fss`
近似平均单散射菲涅尔值，基于 Kulla-Conty 2017 论文的数值拟合。

### 导体菲涅尔

#### `fresnel_conductor_polarized` (float 版本)
计算电介质-导体界面的偏振菲涅尔反射率及反射相移，基于 Born-Wolf《光学原理》第 14.4.1 节。支持 F82 金属模型。

#### `fresnel_conductor_polarized` (Spectrum 版本)
光谱版本，逐分量调用标量版本以降低 GPU 寄存器压力。

#### `fresnel_conductor`
非偏振光在导体表面的菲涅尔反射率。

#### `fresnel_conductor_Fss`
近似导体的平均单散射菲涅尔值，通过 F82 模型拟合实现。

### F82 金属模型

#### `fresnel_f82_B` / `fresnel_f82tint_B`
预计算 F82 模型的 B 项。F82-Tint 变体使用经典 Schlick 模型乘以色调因子。

#### `fresnel_f82`
在给定入射角余弦下求值 F82 金属菲涅尔模型。

#### `fresnel_f82_Fss`
F82 模型的平均单散射菲涅尔解析表达式。

### 折射率转换

#### `ior_from_F0` / `F0_from_ior`
在折射率和垂直入射反射率 F0 之间互转。

#### `schlick_fresnel`
Schlick 近似的 (1-u)^5 项。

#### `interpolate_fresnel_color`
基于实际菲涅尔值在 F0 颜色和白色之间插值，解决低相对折射率下 Schlick 近似不准确的问题。

### 法线修正

#### `ensure_valid_specular_reflection`
当着色法线导致高光反射落入几何表面下方时，将着色法线向几何法线旋转，确保反射方向有效。通过求解四次方程找到最小旋转角。

#### `maybe_ensure_valid_specular_reflection`
条件性调用上述修正。对曲线图元或着色法线与几何法线相同的情况跳过修正。

### 毛发着色

#### `bsdf_principled_hair_albedo_roughness_scale`
计算 Principled Hair 模型的反照率-粗糙度缩放因子。

#### `bsdf_principled_hair_sigma_from_reflectance`
从反射率颜色和方位角粗糙度计算毛发吸收系数 sigma。

#### `bsdf_principled_hair_sigma_from_concentration`
从真黑色素和棕黑色素浓度计算毛发吸收系数。

### 闭包分层

#### `closure_layering_weight`
计算分层闭包中底层闭包的权重。根据顶层反照率估算剩余能量。

### 薄膜虹彩

#### `fresnel_iridescence`
基于 Belcour-Barla 2017 论文实现薄膜干涉效果。通过 Airy 求和计算多层界面间的干涉反射率。

#### `iridescence_lookup_sensitivity` / `iridescence_airy_summation`
虹彩效果的辅助函数：灵敏度函数查找表读取和 Airy 级数求和。

#### `refract_angle`
根据折射角余弦和相对折射率计算折射方向。

## 依赖关系
- **内部头文件**:
  - `kernel/types.h` — 内核类型定义
  - `kernel/util/colorspace.h` — 色彩空间转换
  - `kernel/util/lookup_table.h` — 查找表读取
  - `util/color.h` — 颜色工具
  - `util/types_spectrum.h` — 光谱类型定义
- **被引用**:
  - `src/kernel/svm/closure.h` — SVM 闭包节点
  - `src/kernel/svm/fresnel.h` — SVM 菲涅尔节点
  - `src/kernel/closure/bsdf_burley.h` — Burley 漫反射
  - `src/kernel/closure/bsdf_microfacet.h` — 微表面 BSDF
  - `src/kernel/closure/bsdf_principled_hair_chiang.h` — Chiang 毛发模型
  - `src/kernel/closure/bsdf_principled_hair_huang.h` — Huang 毛发模型

## 实现细节 / 关键算法

### 高光法线修正算法
`ensure_valid_specular_reflection` 的核心是求解一个关于新法线 N' 的 z 分量的四次方程。在由几何法线 Ng 和原始着色法线 N 张成的二维坐标系中，利用 N' 归一化和反射方程的约束条件，将问题简化为一元二次方程求解。根据入射光 X 分量的符号选择正确的根。

### 薄膜虹彩 Airy 求和
虹彩效果基于薄膜上下界面之间多次反射的干涉。关键步骤：
1. 计算上界面（环境-薄膜）和下界面（薄膜-基底）的菲涅尔反射率
2. 计算薄膜内部的光程差 OPD
3. 通过 Airy 级数（截断到 m=3）对不同路径阶差的贡献求和
4. 使用预计算的灵敏度查找表将光程差映射到 RGB 颜色偏移

### 导体菲涅尔相位计算
导体菲涅尔除反射率外还需计算反射相移，以相量（phasor）形式 `exp(i*phi) = {cos(phi), sin(phi)}` 表示。这避免了 atan2 和 cos 的额外计算开销。

## 关联文件
- `src/kernel/closure/bsdf_microfacet.h` — 微表面模型，大量使用本文件的菲涅尔函数
- `src/kernel/closure/bsdf_principled_hair_chiang.h` — 毛发模型，使用毛发吸收系数计算
- `src/kernel/closure/bsdf_principled_hair_huang.h` — 毛发模型，使用毛发吸收系数和透明处理
- `src/kernel/svm/closure.h` — SVM 层级的闭包初始化
