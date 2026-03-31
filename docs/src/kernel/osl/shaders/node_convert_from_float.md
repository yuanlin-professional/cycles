# node_convert_from_float.osl - 浮点数类型转换着色器

## 概述

该着色器实现从 `float`（浮点数）类型到其他数据类型的转换。属于 Cycles 渲染器开放着色语言（OSL）类型转换节点系列之一。浮点数值被广播填充到多分量类型的所有通道。

## 着色器签名

```osl
shader node_convert_from_float(
    float value_float = 0.0,
    output string value_string = "",
    output int value_int = 0,
    output color value_color = 0.0,
    output vector value_vector = vector(0.0, 0.0, 0.0),
    output point value_point = point(0.0, 0.0, 0.0),
    output normal value_normal = normal(0.0, 0.0, 0.0))
```

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| value_float | float | 0.0 | 输入浮点数值，待转换的源数据 |

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| value_string | string | 字符串输出（当前未实现，始终为空字符串） |
| value_int | int | 浮点数截断为整数 |
| value_color | color | 浮点数广播填充为颜色的 RGB 三通道 |
| value_vector | vector | 浮点数广播填充为向量的 XYZ 三分量 |
| value_point | point | 浮点数广播填充为点坐标的 XYZ 三分量 |
| value_normal | normal | 浮点数广播填充为法线的 XYZ 三分量 |

## 实现逻辑

1. **整数转换**：直接将浮点数强制转换（截断）为整数类型。
2. **颜色/向量/点/法线转换**：将同一浮点数值填充到目标类型的所有三个分量中（即 `(f, f, f)` 形式）。
3. **字符串转换**：未实现。

## 对应 SVM 节点

对应 SVM 内核中的 `NODE_CONVERT` 节点，类型为 `SHADER_NODE_CONVERT`，源类型为 `SocketType::FLOAT`。
