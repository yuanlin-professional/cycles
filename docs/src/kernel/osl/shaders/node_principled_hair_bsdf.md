# node_principled_hair_bsdf.osl - 原理化毛发双向散射分布函数(BSDF)着色器

## 概述

该着色器实现了基于物理的毛发光照计算。支持 Huang 和 Chiang 两种毛发散射模型，以及三种颜色参数化方式：直接着色、黑色素浓度和吸收系数。能够模拟真实毛发中的多重散射光路（R、TT、TRT 瓣），支持毛发截面椭圆形比例和随机变化。

## 着色器签名

```osl
shader node_principled_hair_bsdf(color Color = color(0.017513, 0.005763, 0.002059),
                                 float Melanin = 0.8,
                                 float MelaninRedness = 1.0,
                                 float RandomColor = 0.0,
                                 color Tint = 1.0,
                                 color AbsorptionCoefficient = color(0.245531, 0.52, 1.365),
                                 normal Normal = Ng,
                                 string model = "Huang",
                                 string parametrization = "Direct Coloring",
                                 float Offset = radians(2),
                                 float Roughness = 0.3,
                                 float RadialRoughness = 0.3,
                                 float RandomRoughness = 0.0,
                                 float Coat = 0.0,
                                 float IOR = 1.55,
                                 string AttrRandom = "geom:curve_random",
                                 float Random = 0.0,
                                 float AspectRatio = 0.85,
                                 float Rlobe = 1.0,
                                 float TTlobe = 1.0,
                                 float TRTlobe = 1.0,
                                 output closure color BSDF = 0)
```

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| Color | color | (0.017513, 0.005763, 0.002059) | 毛发颜色，用于直接着色模式 |
| Melanin | float | 0.8 | 黑色素浓度，范围 [0, 1]，用于黑色素模式 |
| MelaninRedness | float | 1.0 | 黑色素红度比，控制真黑色素与褐黑色素比例 |
| RandomColor | float | 0.0 | 颜色随机变化强度 |
| Tint | color | 1.0 | 附加色调，用于黑色素模式 |
| AbsorptionCoefficient | color | (0.245531, 0.52, 1.365) | 吸收系数，用于吸收系数模式 |
| Normal | normal | Ng | 法线方向，默认使用几何法线 |
| model | string | "Huang" | 毛发散射模型：`"Huang"` 或 `"Chiang"` |
| parametrization | string | "Direct Coloring" | 颜色参数化方式：`"Direct Coloring"`、`"Melanin concentration"` 或 `"Absorption coefficient"` |
| Offset | float | radians(2) | 毛鳞片倾斜角偏移（弧度） |
| Roughness | float | 0.3 | 纵向粗糙度 |
| RadialRoughness | float | 0.3 | 径向粗糙度 |
| RandomRoughness | float | 0.0 | 粗糙度随机变化强度 |
| Coat | float | 0.0 | 涂层强度，影响 R 瓣粗糙度 |
| IOR | float | 1.55 | 毛发折射率 |
| AttrRandom | string | "geom:curve_random" | 随机值属性名称 |
| Random | float | 0.0 | 外部随机值输入 |
| AspectRatio | float | 0.85 | 毛发截面椭圆宽高比，仅用于 Huang 模型 |
| Rlobe | float | 1.0 | R 瓣（直接反射）强度因子，仅用于 Huang 模型 |
| TTlobe | float | 1.0 | TT 瓣（双次透射）强度因子，仅用于 Huang 模型 |
| TRTlobe | float | 1.0 | TRT 瓣（透射-反射-透射）强度因子，仅用于 Huang 模型 |

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| BSDF | closure color | 毛发散射闭包(Closure)输出 |

## 实现逻辑

### 1. 随机值获取
- 若 Random 端口已连接则使用其值，否则从曲线属性 `geom:curve_random` 获取。

### 2. 粗糙度计算
- 根据随机值和 RandomRoughness 计算粗糙度变化因子：`1 + 2 * (random - 0.5) * RandomRoughness`。
- Coat 参数影响 R 瓣粗糙度：`m0_roughness = 1 - clamp(Coat, 0, 1)`。

### 3. 吸收系数计算（三种参数化方式）
- **吸收系数模式**：直接使用 AbsorptionCoefficient。
- **黑色素浓度模式**：
  - 随机化黑色素浓度。
  - 将线性 [0, 1] 映射到 [0, inf]：`melanin = -log(max(1 - melanin, 0.0001))`。
  - 按 MelaninRedness 分离为真黑色素和褐黑色素。
  - 使用辅助函数 `sigma_from_concentration` 计算吸收系数。
  - 叠加 Tint 对应的吸收系数。
- **直接着色模式**：通过 `sigma_from_reflectance` 从颜色反算吸收系数。

### 4. 模型选择
- **Huang 模型**：使用 `hair_huang` 闭包，支持椭圆截面（AspectRatio）和独立的 R/TT/TRT 瓣控制。
- **Chiang 模型**：使用 `hair_chiang` 闭包，使用纵向和径向粗糙度参数。

### 辅助函数
- `log3(color)`：逐通道计算自然对数。
- `sigma_from_concentration(eumelanin, pheomelanin)`：从黑色素浓度计算吸收系数。
- `sigma_from_reflectance(color, azimuthal_roughness)`：从反射率反算吸收系数。

## 对应 SVM 节点

对应 SVM 中的 `NODE_CLOSURE_BSDF`，闭包类型为 `CLOSURE_BSDF_HAIR_HUANG_ID`（Huang 模型）和 `CLOSURE_BSDF_HAIR_CHIANG_ID`（Chiang 模型）。
