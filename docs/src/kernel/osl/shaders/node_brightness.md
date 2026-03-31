# node_brightness.osl - 亮度/对比度着色器

## 概述

该着色器实现了颜色的亮度（Bright）和对比度（Contrast）调节功能。通过线性变换对输入颜色的每个通道进行调整，并将结果钳制为非负值，防止产生无效的负数颜色值。

## 着色器签名

```osl
shader node_brightness(color ColorIn = 0.8,
                       float Bright = 0.0,
                       float Contrast = 0.0,
                       output color ColorOut = 0.8)
```

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| ColorIn | color | 0.8 | 输入颜色 |
| Bright | float | 0.0 | 亮度偏移量，正值增亮，负值变暗 |
| Contrast | float | 0.0 | 对比度调节量，正值增加对比度，负值降低对比度 |

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| ColorOut | color | 经过亮度和对比度调节后的输出颜色 |

## 实现逻辑

1. 计算缩放系数 `a = 1.0 + Contrast`，作为对比度的乘法因子。
2. 计算偏移量 `b = Bright - Contrast * 0.5`，将亮度偏移和对比度偏移合并。
3. 对输入颜色的 R、G、B 三个通道分别执行线性变换：`ColorOut[i] = max(a * ColorIn[i] + b, 0.0)`。
4. 使用 `max(..., 0.0)` 确保输出值不为负数。

该变换的数学本质是以 0.5 为中心的对比度缩放叠加亮度平移：当 `Contrast > 0` 时，亮于中灰的区域更亮，暗于中灰的区域更暗；`Bright` 则整体平移亮度。

## 对应 SVM 节点

对应 Cycles SVM 中的 `NODE_BRIGHTCONTRAST` 节点，实现逻辑一致。该节点在 Blender 节点编辑器中显示为"亮度/对比度"（Bright/Contrast）节点。
