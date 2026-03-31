# node_volume_coefficients.osl - 体积系数着色器

## 概述

该着色器用于计算体积散射、吸收和发射的闭包组合，是 Cycles 渲染器体积渲染管线的核心节点之一。支持多种散射相位函数（如 Henyey-Greenstein），允许用户分别控制散射、吸收和发射系数。属于开放着色语言（OSL）体积着色器节点。

## 着色器签名

```osl
shader node_volume_coefficients(
    string phase = "Henyey-Greenstein",
    vector AbsorptionCoefficients = vector(1.0, 1.0, 1.0),
    vector ScatterCoefficients = vector(1.0, 1.0, 1.0),
    float Anisotropy = 0.0,
    float IOR = 1.33,
    float Backscatter = 0.1,
    float Alpha = 0.5,
    float Diameter = 20.0,
    vector EmissionCoefficients = vector(0.0, 0.0, 0.0),
    output closure color Volume = 0)
```

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| phase | string | "Henyey-Greenstein" | 散射相位函数类型 |
| AbsorptionCoefficients | vector | (1, 1, 1) | 吸收系数，控制体积对光的吸收强度（每通道独立） |
| ScatterCoefficients | vector | (1, 1, 1) | 散射系数，控制体积对光的散射强度（每通道独立） |
| Anisotropy | float | 0.0 | 各向异性参数，控制散射方向偏好。0 为各向同性，正值为前向散射，负值为后向散射 |
| IOR | float | 1.33 | 折射率，用于 Fournier-Forand 等高级散射模型 |
| Backscatter | float | 0.1 | 后向散射比例 |
| Alpha | float | 0.5 | Alpha 参数，用于 Draine 等散射模型 |
| Diameter | float | 20.0 | 粒子直径参数 |
| EmissionCoefficients | vector | (0, 0, 0) | 发射系数，控制体积的自发光强度（每通道独立） |

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| Volume | closure color | 组合后的体积闭包，包含散射、吸收和发射分量 |

## 实现逻辑

1. **散射闭包创建**：使用 `scatter()` 函数根据指定的相位函数类型和相关参数创建散射闭包。
2. **散射与吸收组合**：将散射闭包乘以散射系数颜色，吸收闭包 `absorption()` 乘以吸收系数颜色，两者相加。
3. **发射添加**：将发射闭包 `emission()` 乘以发射系数颜色后累加到体积闭包。

最终公式：`Volume = ScatterCoeff * scatter(...) + AbsorptionCoeff * absorption() + EmissionCoeff * emission()`

## 对应 SVM 节点

对应 SVM 内核中的体积系数计算节点。与 `SHADER_NODE_SCATTER_VOLUME` 和 `SHADER_NODE_ABSORPTION_VOLUME` 功能相关。
