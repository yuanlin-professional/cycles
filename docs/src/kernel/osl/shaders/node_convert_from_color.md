# node_convert_from_color.osl - 颜色类型转换着色器

## 概述

该着色器实现从 `color`（颜色）类型到其他数据类型的转换。属于 Cycles 渲染器开放着色语言（OSL）类型转换节点系列之一。颜色值通过不同的转换规则映射为浮点数、整数、向量、点、法线和字符串类型。

## 着色器签名

```osl
shader node_convert_from_color(
    color value_color = 0.0,
    output string value_string = "",
    output float value_float = 0.0,
    output int value_int = 0,
    output vector value_vector = vector(0.0, 0.0, 0.0),
    output point value_point = point(0.0, 0.0, 0.0),
    output normal value_normal = normal(0.0, 0.0, 0.0))
```

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| value_color | color | 0.0 | 输入颜色值，待转换的源数据 |

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| value_string | string | 字符串输出（当前未实现，始终为空字符串） |
| value_float | float | 使用 ITU-R BT.709 亮度系数加权计算的标量亮度值 |
| value_int | int | 亮度值的整数截断结果 |
| value_vector | vector | 将颜色 RGB 三通道直接映射为向量 XYZ 分量 |
| value_point | point | 将颜色 RGB 三通道直接映射为点坐标 XYZ 分量 |
| value_normal | normal | 将颜色 RGB 三通道直接映射为法线 XYZ 分量 |

## 实现逻辑

1. **浮点数转换**：使用 ITU-R BT.709 标准的亮度加权系数（R: 0.2126, G: 0.7152, B: 0.0722）对颜色三通道进行加权求和，得到感知亮度值。
2. **整数转换**：与浮点数相同的加权计算后，截断为整数。
3. **向量/点/法线转换**：将颜色的 R、G、B 三通道分别作为 X、Y、Z 分量直接构造对应类型。
4. **字符串转换**：未实现（开放着色语言（OSL）中不支持运行时颜色到字符串转换）。

## 对应 SVM 节点

对应 SVM 内核中的 `NODE_CONVERT` 节点，类型为 `SHADER_NODE_CONVERT`，源类型为 `SocketType::COLOR`。
