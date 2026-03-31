# node_principled_bsdf.osl - 原理化双向散射分布函数(BSDF)着色器

## 概述

该着色器是 Cycles 渲染器中最核心、最综合的表面着色器。基于 Disney 原理化着色模型，将漫反射、次表面散射、金属反射、透射、光泽涂层、绒面光泽和自发光等多种表面特性统一在单个着色器中。通过直观的参数控制，用户可以创建从塑料、金属、玻璃到布料等各种材质。

## 着色器签名

```osl
shader node_principled_bsdf(string distribution = "multi_ggx",
                            string subsurface_method = "random_walk",
                            color BaseColor = color(0.8, 0.8, 0.8),
                            float SubsurfaceWeight = 0.0,
                            float SubsurfaceScale = 0.1,
                            vector SubsurfaceRadius = vector(1.0, 1.0, 1.0),
                            float SubsurfaceIOR = 1.4,
                            float SubsurfaceAnisotropy = 0.0,
                            float Metallic = 0.0,
                            float DiffuseRoughness = 0.0,
                            float SpecularIORLevel = 0.5,
                            color SpecularTint = color(1.0),
                            float Roughness = 0.5,
                            float Anisotropic = 0.0,
                            float AnisotropicRotation = 0.0,
                            float SheenWeight = 0.0,
                            float SheenRoughness = 0.5,
                            color SheenTint = 0.5,
                            float CoatWeight = 0.0,
                            float CoatRoughness = 0.03,
                            float CoatIOR = 1.5,
                            color CoatTint = color(1.0, 1.0, 1.0),
                            float IOR = 1.45,
                            float TransmissionWeight = 0.0,
                            color EmissionColor = 1.0,
                            float EmissionStrength = 0.0,
                            float Alpha = 1.0,
                            float ThinFilmThickness = 0.0,
                            float ThinFilmIOR = 1.33,
                            normal Normal = N,
                            normal CoatNormal = N,
                            normal Tangent = normalize(dPdu),
                            output closure color BSDF = 0)
```

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| distribution | string | "multi_ggx" | 微表面分布模型 |
| subsurface_method | string | "random_walk" | 次表面散射方法，可选 `"random_walk"` 或 `"random_walk_skin"` |
| BaseColor | color | (0.8, 0.8, 0.8) | 基础颜色 |
| SubsurfaceWeight | float | 0.0 | 次表面散射权重，范围 [0, 1] |
| SubsurfaceScale | float | 0.1 | 次表面散射缩放比例 |
| SubsurfaceRadius | vector | (1.0, 1.0, 1.0) | 次表面散射每通道半径 |
| SubsurfaceIOR | float | 1.4 | 次表面散射折射率（仅用于 random_walk_skin 方法） |
| SubsurfaceAnisotropy | float | 0.0 | 次表面散射各向异性 |
| Metallic | float | 0.0 | 金属度，范围 [0, 1]，0 为电介质，1 为金属 |
| DiffuseRoughness | float | 0.0 | 漫反射粗糙度，控制 Oren-Nayar 漫反射 |
| SpecularIORLevel | float | 0.5 | 高光折射率级别，0.5 对应物理 IOR，调整可增强或减弱高光 |
| SpecularTint | color | (1.0) | 高光色调 |
| Roughness | float | 0.5 | 表面粗糙度，范围 [0, 1] |
| Anisotropic | float | 0.0 | 各向异性强度 |
| AnisotropicRotation | float | 0.0 | 各向异性旋转角度 |
| SheenWeight | float | 0.0 | 绒面光泽权重 |
| SheenRoughness | float | 0.5 | 绒面光泽粗糙度 |
| SheenTint | color | 0.5 | 绒面光泽色调 |
| CoatWeight | float | 0.0 | 涂层权重 |
| CoatRoughness | float | 0.03 | 涂层粗糙度 |
| CoatIOR | float | 1.5 | 涂层折射率，最小值为 1.0 |
| CoatTint | color | (1.0, 1.0, 1.0) | 涂层色调，非白色时会对下层产生滤色效果 |
| IOR | float | 1.45 | 材质折射率 |
| TransmissionWeight | float | 0.0 | 透射权重，范围 [0, 1] |
| EmissionColor | color | 1.0 | 自发光颜色 |
| EmissionStrength | float | 0.0 | 自发光强度 |
| Alpha | float | 1.0 | Alpha 透明度，0 为完全透明，1 为完全不透明 |
| ThinFilmThickness | float | 0.0 | 薄膜厚度 |
| ThinFilmIOR | float | 1.33 | 薄膜折射率 |
| Normal | normal | N | 表面法线 |
| CoatNormal | normal | N | 涂层法线 |
| Tangent | normal | normalize(dPdu) | 切线方向 |

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| BSDF | closure color | 原理化着色闭包(Closure)输出 |

## 实现逻辑

着色器采用分层混合架构，按以下步骤构建最终闭包：

### 1. 参数预处理
- 所有权重参数钳制到有效范围。
- 粗糙度进行平方映射。
- 处理各向异性：计算 alpha_x/alpha_y，必要时旋转切线。

### 2. 漫反射/次表面散射层（非金属、非透射部分）
- 当 `metallic < 1` 且 `transmission < 1` 时构建此层。
- 根据 DiffuseRoughness 选择 Lambert 或 Oren-Nayar 漫反射。
- 若 SubsurfaceWeight > 0，使用 `bssrdf` 闭包混合次表面散射效果。
- 若 IOR != 1 或有薄膜，使用 `generalized_schlick_bsdf` 层叠电介质高光反射。
- SpecularIORLevel 参数调整有效 F0：`F0 = F0_from_ior(eta) * 2 * SpecularIORLevel`。

### 3. 透射层
- 当 `metallic < 1` 且 `TransmissionWeight > 0` 时构建。
- 使用 `generalized_schlick_bsdf` 闭包，透射颜色为基础颜色的平方根。
- 处理背面入射时的折射率翻转。
- 与漫反射层按 transmission 权重混合。

### 4. 金属层
- 当 `metallic > 0` 时构建。
- 使用 `microfacet_f82_tint` 闭包，BaseColor 作为 F0，SpecularTint 作为 F82。
- 与前面的混合结果按 metallic 权重混合。

### 5. 自发光
- 当 EmissionStrength != 0 且 EmissionColor != 黑色时，叠加 `emission()` 闭包。

### 6. 涂层层
- 当 CoatWeight > 0 时构建。
- 若 CoatTint 非白色，对下层应用基于角度的吸收滤色。
- 使用 `dielectric_bsdf` 闭包作为涂层，通过 `layer()` 函数层叠在下层之上。

### 7. 绒面光泽层
- 当 SheenWeight > 0 时构建。
- 法线在表面法线和涂层法线之间混合。
- 使用 `sheen` 闭包，通过 `layer()` 函数层叠。

### 8. Alpha 透明度
- 最终结果与 `transparent()` 闭包按 Alpha 值混合。

## 对应 SVM 节点

对应 SVM 中的 `NODE_CLOSURE_BSDF`，闭包类型为 `CLOSURE_BSDF_PRINCIPLED_ID`。该节点在 SVM 中对应一组复杂的闭包组合逻辑。
