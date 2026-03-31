# node_convert_from_normal.osl - 法线类型转换着色器

## 概述

该着色器实现从 `normal`（法线）类型到其他数据类型的转换。属于 Cycles 渲染器开放着色语言（OSL）类型转换节点系列之一。法线的三个分量通过平均值映射为标量类型，或直接重新解释为其他三分量类型。

## 着色器签名

```osl
shader node_convert_from_normal(
    normal value_normal = normal(0.0, 0.0, 0.0),
    output string value_string = "",
    output float value_float = 0.0,
    output int value_int = 0,
    output vector value_vector = vector(0.0, 0.0, 0.0),
    output color value_color = 0.0,
    output point value_point = point(0.0, 0.0, 0.0))
```

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| value_normal | normal | normal(0.0, 0.0, 0.0) | 输入法线值，待转换的源数据 |

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| value_string | string | 字符串输出（当前未实现，始终为空字符串） |
| value_float | float | 法线三分量的算术平均值 |
| value_int | int | 法线三分量算术平均值的整数截断 |
| value_vector | vector | 法线 XYZ 分量直接映射为向量 XYZ 分量 |
| value_color | color | 法线 XYZ 分量直接映射为颜色 RGB 通道 |
| value_point | point | 法线 XYZ 分量直接映射为点坐标 XYZ 分量 |

## 实现逻辑

1. **浮点数转换**：计算法线三个分量之和再乘以 `1/3`，即取算术平均值。
2. **整数转换**：与浮点数相同的平均值计算后截断为整数。
3. **向量/颜色/点转换**：将法线的三个分量逐一赋值给目标类型的三个分量。
4. **字符串转换**：未实现。

## 对应 SVM 节点

对应 SVM 内核中的 `NODE_CONVERT` 节点，类型为 `SHADER_NODE_CONVERT`，源类型为 `SocketType::NORMAL`。
