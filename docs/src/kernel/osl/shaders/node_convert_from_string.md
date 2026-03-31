# node_convert_from_string.osl - 字符串类型转换着色器

## 概述

该着色器实现从 `string`（字符串）类型到其他数据类型的转换。属于 Cycles 渲染器开放着色语言（OSL）类型转换节点系列之一。当前该着色器的函数体为空，所有输出均保持默认值，因为开放着色语言（OSL）运行时不支持字符串到数值类型的动态解析。

## 着色器签名

```osl
shader node_convert_from_string(
    string value_string = "",
    output color value_color = color(0.0, 0.0, 0.0),
    output float value_float = 0.0,
    output int value_int = 0,
    output vector value_vector = vector(0.0, 0.0, 0.0),
    output point value_point = point(0.0, 0.0, 0.0),
    output normal value_normal = normal(0.0, 0.0, 0.0))
```

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| value_string | string | "" | 输入字符串值，待转换的源数据 |

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| value_color | color | 颜色输出（始终为默认值 (0, 0, 0)） |
| value_float | float | 浮点数输出（始终为默认值 0.0） |
| value_int | int | 整数输出（始终为默认值 0） |
| value_vector | vector | 向量输出（始终为默认值 (0, 0, 0)） |
| value_point | point | 点坐标输出（始终为默认值 (0, 0, 0)） |
| value_normal | normal | 法线输出（始终为默认值 (0, 0, 0)） |

## 实现逻辑

着色器函数体为空。字符串到数值类型的转换在开放着色语言（OSL）运行时中不被支持，因此所有输出参数均保留其声明时的默认值。该着色器作为类型转换系统中的占位节点存在，确保节点图中字符串类型的转换连接在语法上是合法的。

## 对应 SVM 节点

对应 SVM 内核中的 `NODE_CONVERT` 节点，类型为 `SHADER_NODE_CONVERT`，源类型为 `SocketType::STRING`。
