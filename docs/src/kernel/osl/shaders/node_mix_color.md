# node_mix_color.osl - 颜色混合着色器（新版）

## 概述

该着色器是新版颜色混合节点，与旧版 `node_mix.osl` 功能类似，但接口和行为有所改进。支持 18 种颜色混合模式，提供独立的输入因子钳制和输出结果钳制控制。复用了 `node_color_blend.h` 中定义的混合函数。属于 Cycles 渲染器开放着色语言（OSL）颜色工具节点。

## 着色器签名

```osl
shader node_mix_color(
    string blend_type = "mix",
    int use_clamp = 0,
    int use_clamp_result = 0,
    float Factor = 0.5,
    color A = 0.0,
    color B = 0.0,
    output color Result = 0.0)
```

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| blend_type | string | "mix" | 混合模式类型，支持与旧版相同的 18 种模式 |
| use_clamp | int | 0 | 是否将输入混合因子钳制到 [0, 1]（否则使用原始值） |
| use_clamp_result | int | 0 | 是否将输出结果钳制到 [0, 1] |
| Factor | float | 0.5 | 混合因子 |
| A | color | 0.0 | 第一个输入颜色（基础色） |
| B | color | 0.0 | 第二个输入颜色（混合色） |

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| Result | color | 混合后的输出颜色 |

## 实现逻辑

1. 根据 `use_clamp` 标志决定是否钳制混合因子：若启用则 `t = clamp(Factor, 0, 1)`，否则 `t = Factor`。
2. 根据 `blend_type` 字符串选择对应的混合函数。`"mix"` 模式直接使用 `mix(A, B, t)` 内置函数，其余模式调用 `node_color_blend.h` 中定义的混合函数。
3. 若 `use_clamp_result` 启用，将结果通过 `clamp()` 钳制到 [0, 1]。

### 与旧版 node_mix 的区别

- 参数命名更清晰（A/B 替代 Color1/Color2，Result 替代 Color）。
- 输入因子钳制和输出结果钳制分别独立控制。
- `"mix"` 模式直接使用 `mix()` 内置函数，而旧版调用 `node_mix_blend()`。
- 混合函数从 `node_color_blend.h` 头文件引入，而非在文件内定义。

## 对应 SVM 节点

对应 SVM 内核中的 `NODE_MIX` 节点，类型为 `SHADER_NODE_MIX`（新版混合节点的颜色模式）。
