# node_scatter_volume.osl - 体积散射着色器

## 概述

该着色器实现了体积(Volume)散射效果。光线穿过体积介质时会发生散射，改变传播方向。支持多种相位函数模型，包括 Henyey-Greenstein、Draine、Rayleigh、Mie 和 Fournier-Forand，适用于不同的散射场景（如云雾、水下、大气等）。

## 着色器签名

```osl
shader node_scatter_volume(string phase = "Henyey-Greenstein",
                           color Color = color(0.8, 0.8, 0.8),
                           float Density = 1.0,
                           float Anisotropy = 0.0,
                           float IOR = 1.33,
                           float Backscatter = 0.1,
                           float Alpha = 0.5,
                           float Diameter = 20.0,
                           output closure color Volume = 0)
```

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| phase | string | "Henyey-Greenstein" | 相位函数类型：`"Henyey-Greenstein"`、`"Draine"`、`"Rayleigh"`、`"Fournier-Forand"`、`"Mie"` |
| Color | color | (0.8, 0.8, 0.8) | 散射颜色 |
| Density | float | 1.0 | 体积密度，最小值为 0 |
| Anisotropy | float | 0.0 | 散射各向异性，范围 [-1, 1]。0 为各向同性，正值前向散射，负值后向散射 |
| IOR | float | 1.33 | 粒子折射率，用于 Fournier-Forand 模型 |
| Backscatter | float | 0.1 | 后向散射比例，用于 Fournier-Forand 模型 |
| Alpha | float | 0.5 | Draine 相位函数的 alpha 参数 |
| Diameter | float | 20.0 | 粒子直径（微米），用于 Mie 模型 |

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| Volume | closure color | 体积散射闭包(Closure)输出 |

## 实现逻辑

1. 通过头文件 `node_scatter.h` 中的 `scatter()` 辅助函数，根据 phase 参数选择对应的散射闭包。
2. 将散射闭包乘以 `Color * max(Density, 0.0)` 作为最终输出。

各相位函数的适用场景：
- **Henyey-Greenstein**：通用散射模型，适用于云雾等。
- **Draine**：天文尘埃散射模型，包含各向异性和 alpha 参数。
- **Rayleigh**：小粒子散射（粒子远小于波长），适用于天空散射。
- **Fournier-Forand**：水下粒子散射模型。
- **Mie**：球形粒子精确散射，使用拟合参数近似，适用于水滴等。

## 对应 SVM 节点

对应 SVM 中的 `NODE_CLOSURE_VOLUME`，闭包类型为 `CLOSURE_VOLUME_HENYEY_GREENSTEIN_ID` 等。
