# node_separate_xyz.osl - 分离 XYZ 着色器

## 概述

该着色器将一个向量值分离为三个独立的浮点数分量（X、Y、Z）。这是 Cycles 渲染器中最基础的向量分解工具节点之一，属于开放着色语言（OSL）实现，是 `node_combine_xyz` 的逆操作。

## 着色器签名

```osl
shader node_separate_xyz(
    vector Vector = 0.8,
    output float X = 0.0,
    output float Y = 0.0,
    output float Z = 0.0)
```

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| Vector | vector | 0.8 | 输入向量，待分离的源数据 |

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| X | float | 向量的 X（第一个）分量 |
| Y | float | 向量的 Y（第二个）分量 |
| Z | float | 向量的 Z（第三个）分量 |

## 实现逻辑

通过数组下标访问向量的三个分量，分别赋值给对应的输出参数：
- `X = Vector[0]`
- `Y = Vector[1]`
- `Z = Vector[2]`

## 对应 SVM 节点

对应 SVM 内核中的 `NODE_SEPARATE_VECTOR` 节点，类型为 `SHADER_NODE_SEPARATE_XYZ`。
