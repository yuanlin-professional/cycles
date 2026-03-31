# node_float_curve.osl - 浮点曲线着色器

## 概述

该着色器通过查找曲线（ramp）将输入浮点值映射为输出浮点值，实现用户自定义的非线性映射关系。支持输入范围缩放和外推，并通过混合因子控制原始值与映射值的混合比例。属于 Cycles 渲染器开放着色语言（OSL）转换工具节点。

## 着色器签名

```osl
shader node_float_curve(
    float ramp[] = {0.0},
    float min_x = 0.0,
    float max_x = 1.0,
    int extrapolate = 1,
    float ValueIn = 0.0,
    float Factor = 0.0,
    output float ValueOut = 0.0)
```

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| ramp | float[] | {0.0} | 曲线查找表数组，存储映射值 |
| min_x | float | 0.0 | 输入值范围的最小值 |
| max_x | float | 1.0 | 输入值范围的最大值 |
| extrapolate | int | 1 | 是否启用外推（当输入超出 [0,1] 范围时） |
| ValueIn | float | 0.0 | 输入浮点值 |
| Factor | float | 0.0 | 混合因子，控制原始值与曲线映射值的混合比例 |

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| ValueOut | float | 经曲线映射和混合后的输出浮点值 |

## 实现逻辑

1. **归一化**：将输入值从 `[min_x, max_x]` 范围映射到 `[0, 1]`：`c = (ValueIn - min_x) / (max_x - min_x)`。
2. **曲线查找**：调用 `rgb_ramp_lookup()` 函数在 ramp 数组中查找对应的映射值（使用单通道模式）。
3. **混合**：使用 `mix()` 函数按 `Factor` 比例在原始输入值和曲线映射值之间插值。`Factor=0` 时完全输出原始值，`Factor=1` 时完全输出映射值。

## 对应 SVM 节点

对应 SVM 内核中的 `NODE_FLOAT_CURVE` 节点，类型为 `SHADER_NODE_FLOAT_CURVE`。
