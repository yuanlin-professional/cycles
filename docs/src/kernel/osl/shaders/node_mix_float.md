# node_mix_float.osl - 浮点数混合着色器

## 概述

该着色器对两个浮点数值进行线性插值混合。属于新版混合节点系列中的浮点数模式，是 Cycles 渲染器开放着色语言（OSL）数学工具节点中最简单的混合操作之一。

## 着色器签名

```osl
shader node_mix_float(
    int use_clamp = 0,
    float Factor = 0.5,
    float A = 0.0,
    float B = 0.0,
    output float Result = 0.0)
```

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| use_clamp | int | 0 | 是否将混合因子钳制到 [0, 1] 范围 |
| Factor | float | 0.5 | 混合因子，控制 A 和 B 之间的插值比例 |
| A | float | 0.0 | 第一个输入值（Factor=0 时的输出值） |
| B | float | 0.0 | 第二个输入值（Factor=1 时的输出值） |

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| Result | float | 混合后的浮点数结果 |

## 实现逻辑

1. 根据 `use_clamp` 标志决定是否钳制混合因子。
2. 使用 `mix(A, B, t)` 内置函数执行线性插值：`Result = A * (1 - t) + B * t`。

## 对应 SVM 节点

对应 SVM 内核中的 `NODE_MIX` 节点，类型为 `SHADER_NODE_MIX`（新版混合节点的浮点数模式）。
