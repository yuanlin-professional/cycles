# node_clamp.osl - 钳制着色器

## 概述

该着色器将输入值钳制（Clamp）到指定的最小值和最大值范围内。支持两种钳制模式：最小最大模式（MinMax）和范围模式（Range）。范围模式在最小值大于最大值时会自动交换上下界。

## 着色器签名

```osl
shader node_clamp(string clamp_type = "minmax",
                  float Value = 1.0,
                  float Min = 0.0,
                  float Max = 1.0,
                  output float Result = 0.0)
```

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| clamp_type | string | "minmax" | 钳制类型。`"minmax"` 为标准模式，`"range"` 为范围模式（自动处理 Min > Max 的情况） |
| Value | float | 1.0 | 待钳制的输入值 |
| Min | float | 0.0 | 范围下界 |
| Max | float | 1.0 | 范围上界 |

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| Result | float | 钳制后的输出值 |

## 实现逻辑

- **MinMax 模式**（`clamp_type == "minmax"`）：直接执行 `clamp(Value, Min, Max)`。如果 `Min > Max`，结果由开放着色语言（OSL）的 `clamp` 行为决定。
- **Range 模式**（`clamp_type == "range"`）：当 `Min > Max` 时，自动交换两个边界，执行 `clamp(Value, Max, Min)`；否则正常执行 `clamp(Value, Min, Max)`。

这使得范围模式更加健壮，无论用户如何设置 Min 和 Max，都能得到合理的结果。

## 对应 SVM 节点

对应 Cycles SVM 中的 `NODE_CLAMP` 节点。该节点在 Blender 节点编辑器中显示为"钳制"（Clamp）节点。
