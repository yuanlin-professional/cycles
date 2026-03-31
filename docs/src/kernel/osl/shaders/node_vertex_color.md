# node_vertex_color.osl - 顶点颜色着色器

## 概述

顶点颜色着色器（Vertex Color Node / Color Attribute Node）用于读取网格顶点上的颜色属性数据。该着色器是开放着色语言（OSL）中的输入类节点，可从指定的颜色图层中获取 RGBA 颜色值。

## 着色器签名

```osl
shader node_vertex_color(
    string bump_offset = "center",
    float bump_filter_width = BUMP_FILTER_WIDTH,
    string layer_name = "",
    output color Color = 0.0,
    output float Alpha = 0.0
)
```

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| bump_offset | string | `"center"` | 凹凸贴图偏移方向，可选 `"center"`、`"dx"`、`"dy"` |
| bump_filter_width | float | `BUMP_FILTER_WIDTH` | 凹凸贴图滤波宽度 |
| layer_name | string | `""` | 顶点颜色图层名称，为空时使用默认图层 |

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| Color | color | 顶点颜色的 RGB 分量 |
| Alpha | float | 顶点颜色的 Alpha 分量 |

## 实现逻辑

1. **图层名称解析**：若 `layer_name` 为空，使用默认属性名 `"geom:vertex_color"`；否则使用指定的图层名称。

2. **属性读取**：通过 `getattribute` 将顶点颜色读取到长度为 4 的浮点数组中，前三个元素构建 `Color`，第四个元素作为 `Alpha`。

3. **凹凸偏移**：属性读取成功后，若 `bump_offset` 为 `"dx"` 或 `"dy"`，对 `Color` 和 `Alpha` 施加微分偏移。

4. **错误处理**：若属性读取失败，输出警告信息 `"Invalid attribute."`。

## 对应 SVM 节点

对应 Cycles SVM（着色器虚拟机）中的 `NODE_VERTEX_COLOR` 节点。
