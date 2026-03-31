# node_displacement.osl - 置换着色器

## 概述

该着色器根据高度值沿法线方向计算表面置换向量。用于在渲染时修改几何体的表面形状，创建凹凸和细节效果。支持物体空间和世界空间两种模式。属于 Cycles 渲染器开放着色语言（OSL）矢量工具节点。

## 着色器签名

```osl
shader node_displacement(
    string space = "object",
    float Height = 0.0,
    float Midlevel = 0.5,
    float Scale = 1.0,
    normal Normal = N,
    output vector Displacement = vector(0.0, 0.0, 0.0))
```

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| space | string | "object" | 置换空间，支持 "object"（物体空间）和 "world"（世界空间） |
| Height | float | 0.0 | 高度值，表示置换的距离 |
| Midlevel | float | 0.5 | 中间基准值，高度值减去此值为实际偏移 |
| Scale | float | 1.0 | 置换的整体缩放因子 |
| Normal | normal | N | 置换方向的法线，默认使用着色点法线 |

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| Displacement | vector | 计算得到的置换向量 |

## 实现逻辑

1. 以输入法线作为初始置换方向。
2. 若空间模式为 "object"，将法线从世界空间变换到物体空间。
3. 对法线进行归一化，然后乘以 `(Height - Midlevel) * Scale` 计算置换量。
4. 若空间模式为 "object"，将置换向量从物体空间变换回世界空间。

实际置换公式为：`normalize(N_space) * (Height - Midlevel) * Scale`

## 对应 SVM 节点

对应 SVM 内核中的 `NODE_DISPLACEMENT` 节点，类型为 `SHADER_NODE_DISPLACEMENT`。
