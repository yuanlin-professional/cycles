# node_combine_xyz.osl - 合并 XYZ 着色器

## 概述

该着色器将三个独立的浮点数分量（X、Y、Z）合并为一个向量值。这是 Cycles 渲染器中最基础的向量构造工具节点之一，属于开放着色语言（OSL）实现。

## 着色器签名

```osl
shader node_combine_xyz(
    float X = 0.0,
    float Y = 0.0,
    float Z = 0.0,
    output vector Vector = 0.8)
```

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| X | float | 0.0 | 向量的 X 分量 |
| Y | float | 0.0 | 向量的 Y 分量 |
| Z | float | 0.0 | 向量的 Z 分量 |

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| Vector | vector | 由 X、Y、Z 分量合并而成的向量 |

## 实现逻辑

直接使用 `vector(X, Y, Z)` 构造函数将三个标量分量组合为一个向量。逻辑非常简单直接。

## 对应 SVM 节点

对应 SVM 内核中的 `NODE_COMBINE_VECTOR` 节点，类型为 `SHADER_NODE_COMBINE_XYZ`。
