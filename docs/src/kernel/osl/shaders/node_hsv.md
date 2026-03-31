# node_hsv.osl - 色相/饱和度/明度着色器

## 概述

该着色器在 HSV（色相-饱和度-明度）色彩空间中调整颜色。输入颜色先从 RGB 转换到 HSV 空间，分别对色相、饱和度和明度进行调整，再转换回 RGB 空间，最后通过混合因子与原始颜色混合。

## 着色器签名

```osl
shader node_hsv(float Hue = 0.5,
                float Saturation = 1.0,
                float Value = 1.0,
                float Fac = 0.5,
                color ColorIn = 0.0,
                output color ColorOut = 0.0)
```

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| Hue | float | 0.5 | 色相偏移量。0.5 表示无偏移，小于 0.5 向负方向旋转，大于 0.5 向正方向旋转 |
| Saturation | float | 1.0 | 饱和度缩放因子。1.0 为原始饱和度，0.0 为完全去饱和 |
| Value | float | 1.0 | 明度缩放因子。1.0 为原始明度 |
| Fac | float | 0.5 | 混合因子，控制原始颜色与调整后颜色的混合比例（0.0 = 原色，1.0 = 完全调整） |
| ColorIn | color | 0.0 | 输入颜色 |

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| ColorOut | color | 经过 HSV 调整并混合后的输出颜色 |

## 实现逻辑

1. 调用 `rgb_to_hsv()` 将输入颜色从 RGB 转换到 HSV 空间。
2. **色相调整**：`H = fract(H + Hue + 0.5)`，使用 `fract()` 保证色相值在 [0, 1) 范围内循环。加 0.5 的偏移使得 `Hue = 0.5` 时色相不变。
3. **饱和度调整**：`S = clamp(S * Saturation, 0.0, 1.0)`，乘以缩放因子并钳制到 [0, 1]。
4. **明度调整**：`V *= Value`，直接乘以缩放因子。
5. 调用 `hsv_to_rgb()` 将调整后的 HSV 转换回 RGB。
6. 对转换结果的每个通道执行 `max(..., 0.0)` 钳制，防止过饱和导致的负值。
7. 使用 `mix(ColorIn, Color, Fac)` 将原始颜色与调整后颜色按因子混合。

## 对应 SVM 节点

对应 Cycles SVM 中的 `NODE_HSV` 节点。该节点在 Blender 节点编辑器中显示为"色相/饱和度/明度"（Hue/Saturation/Value）节点。依赖 `node_color.h` 中定义的 `rgb_to_hsv()` 和 `hsv_to_rgb()` 转换函数。
