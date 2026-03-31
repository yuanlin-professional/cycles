# node_uv_map.osl - UV 贴图着色器

## 概述

UV 贴图着色器（UV Map Node）用于获取网格的 UV 纹理坐标。该着色器是开放着色语言（OSL）中的输入类节点，支持从指定的 UV 图层或复制体（Dupli）UV 中读取坐标数据。

## 着色器签名

```osl
shader node_uv_map(
    int from_dupli = 0,
    string attribute = "",
    string bump_offset = "center",
    float bump_filter_width = BUMP_FILTER_WIDTH,
    output point UV = point(0.0, 0.0, 0.0)
)
```

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| from_dupli | int | `0` | 是否从复制体（实例化对象）获取 UV，1 为是 |
| attribute | string | `""` | UV 图层的属性名称，为空时使用默认 UV 图层 |
| bump_offset | string | `"center"` | 凹凸贴图偏移方向，可选 `"center"`、`"dx"`、`"dy"` |
| bump_filter_width | float | `BUMP_FILTER_WIDTH` | 凹凸贴图滤波宽度 |

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| UV | point | UV 纹理坐标，以三维点形式输出（U, V, 0） |

## 实现逻辑

1. **UV 来源选择**：
   - 若 `from_dupli` 为真，通过 `getattribute("geom:dupli_uv", UV)` 从复制体获取 UV。
   - 若 `from_dupli` 为假且 `attribute` 为空，通过 `getattribute("geom:uv", UV)` 获取默认 UV。
   - 若 `from_dupli` 为假且 `attribute` 非空，通过 `getattribute(attribute, UV)` 获取指定图层的 UV。

2. **凹凸偏移**：当 `bump_offset` 为 `"dx"` 或 `"dy"` 时，对 UV 坐标施加微分偏移。注意：从复制体获取的 UV 不进行凹凸偏移处理。

## 对应 SVM 节点

对应 Cycles SVM（着色器虚拟机）中的 `NODE_UV_MAP` 节点。
