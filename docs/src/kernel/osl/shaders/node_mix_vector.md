# node_mix_vector.osl - 向量混合着色器

## 概述

该着色器对两个向量值使用统一因子进行线性插值混合。属于新版混合节点系列中的向量模式（均匀混合），混合因子为标量，同时作用于向量的所有分量。属于 Cycles 渲染器开放着色语言（OSL）矢量工具节点。

## 着色器签名

```osl
shader node_mix_vector(
    int use_clamp = 0,
    float Factor = 0.5,
    vector A = 0.0,
    vector B = 0.0,
    output vector Result = 0.0)
```

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| use_clamp | int | 0 | 是否将混合因子钳制到 [0, 1] 范围 |
| Factor | float | 0.5 | 混合因子（标量），同时控制所有分量的插值比例 |
| A | vector | 0.0 | 第一个输入向量（Factor=0 时的输出值） |
| B | vector | 0.0 | 第二个输入向量（Factor=1 时的输出值） |

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| Result | vector | 混合后的向量结果 |

## 实现逻辑

1. 根据 `use_clamp` 标志决定是否钳制混合因子。
2. 使用 `mix(A, B, t)` 内置函数执行向量的逐分量线性插值。

与 `node_mix_vector_non_uniform` 的区别在于：本着色器使用标量因子对所有分量执行均匀混合。

## 对应 SVM 节点

对应 SVM 内核中的 `NODE_MIX` 节点，类型为 `SHADER_NODE_MIX`（新版混合节点的向量均匀模式）。
