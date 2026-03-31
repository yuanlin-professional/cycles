# node_absorption_volume.osl - 体积吸收着色器

## 概述

该着色器实现了体积(Volume)吸收效果。光线穿过体积介质时会被吸收，导致光线强度按指数衰减。颜色参数控制哪些波长的光被吸收（实际吸收的是颜色的互补色）。常用于模拟烟雾、有色液体等对光线的吸收。

## 着色器签名

```osl
shader node_absorption_volume(color Color = color(0.8, 0.8, 0.8),
                              float Density = 1.0,
                              output closure color Volume = 0)
```

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| Color | color | (0.8, 0.8, 0.8) | 体积颜色。白色表示无吸收，越深的颜色表示越强的吸收 |
| Density | float | 1.0 | 体积密度，最小值为 0 |

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| Volume | closure color | 体积吸收闭包(Closure)输出 |

## 实现逻辑

吸收系数的计算公式为 `(1.0 - Color) * max(Density, 0.0)`。

- `(1.0 - Color)` 将颜色转换为吸收系数：白色(1,1,1) 对应零吸收，黑色(0,0,0) 对应最大吸收。
- 密度值钳制为非负值后作为吸收强度的缩放因子。
- 最终使用开放着色语言(OSL)内置的 `absorption()` 闭包。

## 对应 SVM 节点

对应 SVM 中的 `NODE_CLOSURE_VOLUME`，闭包类型为 `CLOSURE_VOLUME_ABSORPTION_ID`。
