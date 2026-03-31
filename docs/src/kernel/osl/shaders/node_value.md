# node_value.osl - 常量值着色器

## 概述

该着色器是一个简单的常量传递节点，将输入的浮点数、向量和颜色值直接传递到输出端口。在 Blender 节点编辑器中，它作为 "值" 节点提供用户设定的常量值供其他节点使用。属于 Cycles 渲染器开放着色语言（OSL）输入节点。

## 着色器签名

```osl
shader node_value(
    float value_value = 0.0,
    vector vector_value = vector(0.0, 0.0, 0.0),
    color color_value = 0.0,
    output float Value = 0.0,
    output vector Vector = vector(0.0, 0.0, 0.0),
    output color Color = 0.0)
```

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| value_value | float | 0.0 | 输入浮点数常量值 |
| vector_value | vector | (0, 0, 0) | 输入向量常量值 |
| color_value | color | 0.0 | 输入颜色常量值 |

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| Value | float | 传递的浮点数值 |
| Vector | vector | 传递的向量值 |
| Color | color | 传递的颜色值 |

## 实现逻辑

直接将三个输入值逐一赋值给对应的输出参数，不进行任何计算或变换。该着色器的核心作用是在着色器图中提供用户指定的常量值。

## 对应 SVM 节点

对应 SVM 内核中的 `NODE_VALUE_F` / `NODE_VALUE_V` 节点，类型为 `SHADER_NODE_VALUE`。
