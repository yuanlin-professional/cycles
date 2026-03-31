# node_glossy_bsdf.osl - 光泽双向散射分布函数(BSDF)着色器

## 概述

该着色器实现了光泽（高光）反射表面的光照计算，基于微表面理论。支持各向异性反射和多种微表面分布模型，包括 GGX 和多重散射 GGX（Multiscatter GGX）。通过各向异性参数和旋转角度可以控制反射的方向性拉伸效果。

## 着色器签名

```osl
shader node_glossy_bsdf(color Color = 0.8,
                        string distribution = "ggx",
                        float Roughness = 0.2,
                        float Anisotropy = 0.0,
                        float Rotation = 0.0,
                        normal Normal = N,
                        normal Tangent = 0.0,
                        output closure color BSDF = 0)
```

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| Color | color | 0.8 | 反射颜色，负值会被钳制为 0 |
| distribution | string | "ggx" | 微表面分布模型，可选 `"ggx"` 或 `"Multiscatter GGX"` |
| Roughness | float | 0.2 | 表面粗糙度，范围 [0, 1]，内部会进行平方映射 |
| Anisotropy | float | 0.0 | 各向异性强度，范围 [-0.99, 0.99]，0 为各向同性 |
| Rotation | float | 0.0 | 各向异性旋转角度，值域 [0, 1] 映射到 [0, 2*pi] |
| Normal | normal | N | 表面法线方向 |
| Tangent | normal | 0.0 | 切线方向，用于各向异性反射 |

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| BSDF | closure color | 光泽反射闭包(Closure)输出 |

## 实现逻辑

1. **颜色处理**：将基础颜色钳制为非负值。
2. **粗糙度映射**：将粗糙度钳制到 [0, 1] 范围后进行平方映射（`roughness = roughness * roughness`），使感知上更线性。
3. **各向异性处理**：
   - 若各向异性值接近零（`<= 1e-4`），U 和 V 方向粗糙度相同。
   - 若各向异性不为零，先根据旋转角度绕法线旋转切线向量，然后根据各向异性值的正负号分别计算 U/V 方向粗糙度。负值各向异性交换 U/V 方向的拉伸关系。
4. **闭包选择**：
   - `"Multiscatter GGX"` 模式：使用 `microfacet_multi_ggx_aniso` 闭包，支持微表面间多重散射能量补偿。
   - 其他模式：使用通用 `microfacet` 闭包，折射率设为 0（纯反射），不进行折射。

## 对应 SVM 节点

对应 SVM 中的 `NODE_CLOSURE_BSDF`，闭包类型为 `CLOSURE_BSDF_MICROFACET_GGX_ID` 和 `CLOSURE_BSDF_MICROFACET_MULTI_GGX_ID`。
