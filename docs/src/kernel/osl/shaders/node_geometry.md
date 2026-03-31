# node_geometry.osl - 几何信息着色器

## 概述

几何信息着色器（Geometry Node）用于获取当前着色点的几何属性信息，包括位置、法线、切线、参数坐标、背面检测、尖锐度以及每孤岛随机值等。该着色器是开放着色语言（OSL）中的输入类节点，为其他着色器提供几何数据源。

## 着色器签名

```osl
shader node_geometry(
    string bump_offset = "center",
    float bump_filter_width = BUMP_FILTER_WIDTH,
    output point Position = point(0.0, 0.0, 0.0),
    output normal Normal = normal(0.0, 0.0, 0.0),
    output normal Tangent = normal(0.0, 0.0, 0.0),
    output normal TrueNormal = normal(0.0, 0.0, 0.0),
    output vector Incoming = vector(0.0, 0.0, 0.0),
    output point Parametric = point(0.0, 0.0, 0.0),
    output float Backfacing = 0.0,
    output float Pointiness = 0.0,
    output float RandomPerIsland = 0.0
)
```

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| bump_offset | string | `"center"` | 凹凸贴图偏移方向，可选 `"center"`、`"dx"`、`"dy"` |
| bump_filter_width | float | `BUMP_FILTER_WIDTH` | 凹凸贴图滤波宽度，由 `stdcycles.h` 中的宏定义 |

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| Position | point | 着色点的世界空间位置坐标（`P`） |
| Normal | normal | 着色点的插值法线方向（`N`） |
| Tangent | normal | 着色点的切线方向，优先从生成坐标创建球形切线，曲线/点云则使用表面导数 |
| TrueNormal | normal | 着色点的几何法线方向（`Ng`），未经插值的真实面法线 |
| Incoming | vector | 入射光线方向（`I`） |
| Parametric | point | 三角形参数坐标 `(1-u-v, u, 0)` |
| Backfacing | float | 背面标志，1.0 表示从背面观察，0.0 表示从正面观察 |
| Pointiness | float | 顶点尖锐度，用于检测边缘和凸起区域 |
| RandomPerIsland | float | 每个网格孤岛的随机值，取值范围 [0, 1] |

## 实现逻辑

1. **基本几何信息**：直接从开放着色语言（OSL）内建全局变量获取位置 `P`、法线 `N`、几何法线 `Ng`、入射方向 `I`，并通过重心坐标 `(u, v)` 计算参数坐标。

2. **凹凸偏移处理**：当 `bump_offset` 为 `"dx"` 或 `"dy"` 时，对 `Position` 和 `Parametric` 施加微分偏移，用于凹凸贴图计算。

3. **切线计算**：
   - 检测当前几何体是否为曲线（`geom:is_curve`）或点云（`geom:is_point`）。
   - 对于普通网格，如果存在生成坐标（`geom:generated`），则基于生成坐标创建球形切线，通过物体空间到世界空间的变换和叉乘计算得到。
   - 对于曲线或无生成坐标的情况，使用表面参数导数 `dPdu` 的归一化值作为切线。

4. **尖锐度与孤岛随机值**：通过 `getattribute` 获取 `geom:pointiness` 和 `geom:random_per_island`，尖锐度同样支持凹凸偏移。

## 对应 SVM 节点

对应 Cycles SVM（着色器虚拟机）中的 `NODE_GEOMETRY` 和 `NODE_GEOMETRY_BUMP_DX` / `NODE_GEOMETRY_BUMP_DY` 节点。在 SVM 路径中，各输出分别通过独立的字节码指令实现。
