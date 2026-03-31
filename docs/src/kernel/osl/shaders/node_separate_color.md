# node_separate_color.osl - 分离颜色着色器

## 概述

该着色器将一个颜色值分离为三个独立的浮点数分量。支持多种颜色空间模式（RGB、HSV、HSL），允许用户提取不同颜色模型下的各通道值。属于 Cycles 渲染器开放着色语言（OSL）转换工具节点，是 `node_combine_color` 的逆操作。

## 着色器签名

```osl
shader node_separate_color(
    string color_type = "rgb",
    color Color = 0.8,
    output float Red = 0.0,
    output float Green = 0.0,
    output float Blue = 0.0)
```

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| color_type | string | "rgb" | 颜色空间类型，支持 "rgb"、"hsv"、"hsl" |
| Color | color | 0.8 | 输入的 RGB 颜色值 |

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| Red | float | 第一个分量（RGB 模式下为红色，HSV 模式下为色相，HSL 模式下为色相） |
| Green | float | 第二个分量（RGB 模式下为绿色，HSV 模式下为饱和度，HSL 模式下为饱和度） |
| Blue | float | 第三个分量（RGB 模式下为蓝色，HSV 模式下为明度，HSL 模式下为亮度） |

## 实现逻辑

1. 根据 `color_type` 参数选择颜色空间转换方式：
   - `"rgb"`：直接使用输入颜色，无需转换。
   - `"hsv"`：调用 `rgb_to_hsv()` 将 RGB 转换为 HSV 颜色空间。
   - `"hsl"`：调用 `rgb_to_hsl()` 将 RGB 转换为 HSL 颜色空间。
2. 若 `color_type` 不在支持列表中，输出警告 "Unknown color space!"。
3. 将转换后颜色的三个通道分别赋值给 Red、Green、Blue 输出。

## 对应 SVM 节点

对应 SVM 内核中的 `NODE_SEPARATE_COLOR` 节点，类型为 `SHADER_NODE_SEPARATE_COLOR`。
