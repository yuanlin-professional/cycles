# node_subsurface_scattering.osl - 次表面散射着色器

## 概述

该着色器实现了次表面散射（Subsurface Scattering，SSS）效果，模拟光线进入半透明材质内部后在材质中散射并从不同位置射出的现象。常用于模拟皮肤、蜡烛、大理石、牛奶等半透明材质。支持随机游走等多种散射方法。

## 着色器签名

```osl
shader node_subsurface_scattering(color Color = 0.8,
                                  float Scale = 1.0,
                                  vector Radius = vector(0.1, 0.1, 0.1),
                                  float IOR = 1.4,
                                  float Roughness = 1.0,
                                  float Anisotropy = 0.0,
                                  string method = "random_walk",
                                  normal Normal = N,
                                  output closure color BSSRDF = 0)
```

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| Color | color | 0.8 | 次表面散射颜色，负值被钳制为 0 |
| Scale | float | 1.0 | 散射半径的全局缩放因子 |
| Radius | vector | (0.1, 0.1, 0.1) | 每个颜色通道（RGB）的散射半径 |
| IOR | float | 1.4 | 材质折射率 |
| Roughness | float | 1.0 | 表面粗糙度，范围 [0, 1] |
| Anisotropy | float | 0.0 | 散射各向异性 |
| method | string | "random_walk" | 次表面散射方法，如 `"random_walk"`、`"random_walk_skin"` |
| Normal | normal | N | 表面法线方向 |

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| BSSRDF | closure color | 次表面散射闭包(Closure)输出（BSSRDF） |

## 实现逻辑

1. **颜色处理**：将颜色钳制为非负值。
2. **半径计算**：有效散射半径为 `Scale * Radius`。
3. **闭包构建**：使用 `bssrdf` 闭包，传入散射方法、法线、有效半径和基础颜色，以及可选的 IOR、各向异性和粗糙度参数。

注意输出变量名为 `BSSRDF` 而非 `BSDF`，反映了次表面散射使用的是双向散射表面反射分布函数（BSSRDF）而非普通的双向散射分布函数(BSDF)。

## 对应 SVM 节点

对应 SVM 中的 `NODE_CLOSURE_BSDF`，闭包类型为 `CLOSURE_BSSRDF_RANDOM_WALK_ID` 等。
