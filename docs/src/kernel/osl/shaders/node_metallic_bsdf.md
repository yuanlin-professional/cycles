# node_metallic_bsdf.osl - 金属双向散射分布函数(BSDF)着色器

## 概述

该着色器实现了金属材质的光照计算。支持两种菲涅耳模型：F82 色调映射模型和基于物理的导体菲涅耳模型（使用复折射率）。支持各向异性反射和薄膜干涉效果。F82 模型通过基础颜色和边缘色调提供艺术化控制，而导体模型则使用真实的折射率（IOR）和消光系数（Extinction）。

## 着色器签名

```osl
shader node_metallic_bsdf(color BaseColor = color(0.617, 0.577, 0.540),
                          color EdgeTint = color(0.695, 0.726, 0.770),
                          vector IOR = vector(2.757, 2.513, 2.231),
                          vector Extinction = vector(3.867, 3.404, 3.009),
                          string distribution = "multi_ggx",
                          string fresnel_type = "f82",
                          float Roughness = 0.5,
                          float Anisotropy = 0.0,
                          float Rotation = 0.0,
                          float ThinFilmThickness = 0.0,
                          float ThinFilmIOR = 1.33,
                          normal Normal = N,
                          normal Tangent = 0.0,
                          output closure color BSDF = 0)
```

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| BaseColor | color | (0.617, 0.577, 0.540) | 金属基础颜色（法线入射反射率 F0），默认值对应铜 |
| EdgeTint | color | (0.695, 0.726, 0.770) | 边缘色调（82 度入射角处的反射率 F82），用于 F82 模型 |
| IOR | vector | (2.757, 2.513, 2.231) | 每通道折射率，用于导体菲涅耳模型 |
| Extinction | vector | (3.867, 3.404, 3.009) | 每通道消光系数 k，用于导体菲涅耳模型 |
| distribution | string | "multi_ggx" | 微表面分布模型 |
| fresnel_type | string | "f82" | 菲涅耳类型：`"f82"` 为 F82 色调模型，其他值使用导体菲涅耳 |
| Roughness | float | 0.5 | 表面粗糙度，范围 [0, 1]，内部进行平方映射 |
| Anisotropy | float | 0.0 | 各向异性强度，范围 [0, 1] |
| Rotation | float | 0.0 | 各向异性旋转角度，[0, 1] 映射到 [0, 2*pi] |
| ThinFilmThickness | float | 0.0 | 薄膜厚度 |
| ThinFilmIOR | float | 1.33 | 薄膜折射率 |
| Normal | normal | N | 表面法线方向 |
| Tangent | normal | 0.0 | 切线方向 |

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| BSDF | closure color | 金属反射闭包(Closure)输出 |

## 实现逻辑

1. **粗糙度映射**：将粗糙度钳制到 [0, 1] 后平方映射，初始化 alpha_x 和 alpha_y 相同。
2. **各向异性处理**：当各向异性大于 0 时，通过 `aspect = sqrt(1 - Anisotropy * 0.9)` 计算宽高比，分别缩放 alpha_x 和 alpha_y。若有旋转则绕法线旋转切线。
3. **菲涅耳模型选择**：
   - **F82 模型**（`fresnel_type == "f82"`）：将 BaseColor 作为 F0、EdgeTint 作为 F82，使用 `microfacet_f82_tint` 闭包。颜色值钳制在 [0, 1]。
   - **导体模型**：使用 `conductor_bsdf` 闭包，传入每通道 IOR 和消光系数，进行物理精确的导体菲涅耳计算。
4. 两种模型均支持薄膜干涉参数。

## 对应 SVM 节点

对应 SVM 中的 `NODE_CLOSURE_BSDF`，闭包类型为 `CLOSURE_BSDF_MICROFACET_MULTI_GGX_ID`（F82 模型）和 `CLOSURE_BSDF_METAL_ID`（导体模型）。
