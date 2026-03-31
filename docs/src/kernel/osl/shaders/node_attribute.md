# node_attribute.osl - 属性着色器

## 概述

属性着色器（Attribute Node）用于从几何体上读取指定名称的自定义属性。该着色器是开放着色语言（OSL）中的输入类节点，可以访问顶点属性、面属性或其他自定义数据层，并以颜色、向量、浮点和透明度多种形式输出。

## 着色器签名

```osl
shader node_attribute(
    string bump_offset = "center",
    float bump_filter_width = BUMP_FILTER_WIDTH,
    string name = "",
    output point Vector = point(0.0, 0.0, 0.0),
    output color Color = 0.0,
    output float Fac = 0.0,
    output float Alpha = 0.0
)
```

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| bump_offset | string | `"center"` | 凹凸贴图偏移方向，可选 `"center"`、`"dx"`、`"dy"` |
| bump_filter_width | float | `BUMP_FILTER_WIDTH` | 凹凸贴图滤波宽度 |
| name | string | `""` | 要读取的属性名称 |

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| Vector | point | 属性值以向量/点形式输出 |
| Color | color | 属性值以颜色形式输出（RGB 三个分量） |
| Fac | float | 属性值以标量形式输出（取属性的平均值或第一分量） |
| Alpha | float | 属性的第四分量（Alpha 通道），用于 RGBA 类型属性 |

## 实现逻辑

1. **属性读取**：使用 `getattribute(name, data)` 读取名为 `name` 的属性到长度为 4 的浮点数组中。

2. **回退处理**：如果属性读取失败且属性名为 `"geom:generated"`，则回退到物体空间坐标（`transform("object", P)`）作为替代值，并将 `Alpha` 设为 1.0。

3. **正常路径**：属性读取成功时，使用 `getattribute(name, Fac)` 获取标量值，从数组的前三个元素构建 `Color`。

4. **输出赋值**：`Vector` 从 `Color` 转换而来，`Alpha` 取数组的第四个元素。

5. **凹凸偏移**：当 `bump_offset` 为 `"dx"` 或 `"dy"` 时，对所有输出施加对应的微分偏移（`Dx` 或 `Dy`），用于凹凸贴图计算。

## 对应 SVM 节点

对应 Cycles SVM（着色器虚拟机）中的 `NODE_ATTR` 和 `NODE_ATTR_BUMP_DX` / `NODE_ATTR_BUMP_DY` 节点。
