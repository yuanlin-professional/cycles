# node_output_displacement.osl - 置换输出着色器

## 概述

置换输出着色器（Displacement Output Node）是开放着色语言（OSL）中的输出类节点，用于将计算好的置换向量应用到着色点的位置上，实现几何表面的位移变形效果。

## 着色器签名

```osl
displacement node_output_displacement(
    vector Displacement = 0.0
)
```

注意：该着色器类型为 `displacement`，而非通用 `shader`。

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| Displacement | vector | `0.0`（零向量） | 置换向量，定义表面点沿各方向的位移量 |

## 输出参数

无显式输出参数。该着色器直接修改开放着色语言（OSL）的内建全局变量 `P`。

## 实现逻辑

将置换向量直接叠加到着色点位置上：

```osl
P += Displacement;
```

`P` 是开放着色语言（OSL）中预定义的着色点位置全局变量。在置换着色器中，修改 `P` 会使渲染器重新计算该点的实际位置和法线，从而产生真实的几何细节。

与凹凸贴图（Bump Mapping）不同，置换着色器会真正改变几何体的表面位置，可以影响轮廓线和阴影。

## 对应 SVM 节点

对应 Cycles SVM（着色器虚拟机）中的 `NODE_DISPLACEMENT` 节点。在 SVM 中，置换结果通过修改着色点的位置属性来实现。
