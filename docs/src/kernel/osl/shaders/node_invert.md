# node_invert.osl - 反转着色器

## 概述

该着色器对输入颜色执行反转操作（即计算补色），并通过混合因子控制反转程度。反转操作将每个颜色通道的值从 `c` 变为 `1 - c`，常用于创建底片效果或颜色反转效果。

## 着色器签名

```osl
shader node_invert(float Fac = 1.0,
                   color ColorIn = 0.8,
                   output color ColorOut = 0.8)
```

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| Fac | float | 1.0 | 混合因子。0.0 = 输出原色，1.0 = 完全反转 |
| ColorIn | color | 0.8 | 输入颜色 |

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| ColorOut | color | 反转并混合后的输出颜色 |

## 实现逻辑

1. 计算反转颜色：`ColorInv = color(1.0) - ColorIn`，即对 R、G、B 每个通道执行 `1.0 - c`。
2. 使用 `mix(ColorIn, ColorInv, Fac)` 在原始颜色与反转颜色之间线性插值。
   - `Fac = 0.0`：输出为原始颜色。
   - `Fac = 1.0`：输出为完全反转的颜色。
   - `0 < Fac < 1`：输出为两者的线性混合。

## 对应 SVM 节点

对应 Cycles SVM 中的 `NODE_INVERT` 节点。该节点在 Blender 节点编辑器中显示为"反转"（Invert）节点。
