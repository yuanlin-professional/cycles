# node_combine_color.osl - 合并颜色着色器

## 概述

该着色器将三个独立的浮点数分量合并为一个颜色值。支持多种颜色空间模式（RGB、HSV、HSL），允许用户在不同颜色模型下构造颜色。属于 Cycles 渲染器开放着色语言（OSL）转换工具节点。

## 着色器签名

```osl
shader node_combine_color(
    string color_type = "rgb",
    float Red = 0.0,
    float Green = 0.0,
    float Blue = 0.0,
    output color Color = 0.8)
```

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| color_type | string | "rgb" | 颜色空间类型，支持 "rgb"、"hsv"、"hsl" |
| Red | float | 0.0 | 第一个分量（RGB 模式下为红色，HSV 模式下为色相，HSL 模式下为色相） |
| Green | float | 0.0 | 第二个分量（RGB 模式下为绿色，HSV 模式下为饱和度，HSL 模式下为饱和度） |
| Blue | float | 0.0 | 第三个分量（RGB 模式下为蓝色，HSV 模式下为明度，HSL 模式下为亮度） |

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| Color | color | 合并后的颜色值（RGB 空间） |

## 实现逻辑

1. 检查 `color_type` 是否为支持的颜色空间之一（"rgb"、"hsv"、"hsl"）。
2. 若合法，使用开放着色语言（OSL）内置的 `color(color_type, ...)` 构造函数创建颜色。该函数会根据指定的颜色空间将三个分量自动转换为 RGB 颜色。
3. 若 `color_type` 不在支持列表中，输出警告 "Unknown color space!"。

## 对应 SVM 节点

对应 SVM 内核中的 `NODE_COMBINE_COLOR` 节点，类型为 `SHADER_NODE_COMBINE_COLOR`。
