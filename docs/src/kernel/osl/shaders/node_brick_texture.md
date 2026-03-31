# node_brick_texture.osl - 砖墙纹理着色器

## 概述

该开放着色语言(OSL)着色器节点生成程序化砖墙纹理图案。通过模拟砖块与灰缝的排列，支持砖块偏移、挤压变形、灰缝平滑过渡以及砖块颜色随机着色等功能，广泛用于建筑材质的程序化生成。

## 着色器签名

```osl
shader node_brick_texture(
    int use_mapping = 0,
    matrix mapping = matrix(0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0),
    float offset = 0.5,
    int offset_frequency = 2,
    float squash = 1.0,
    int squash_frequency = 1,
    point Vector = P,
    color Color1 = 0.2,
    color Color2 = 0.8,
    color Mortar = 0.0,
    float Scale = 5.0,
    float MortarSize = 0.02,
    float MortarSmooth = 0.0,
    float Bias = 0.0,
    float BrickWidth = 0.5,
    float RowHeight = 0.25,
    output float Fac = 0.0,
    output color Color = 0.2)
```

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| use_mapping | int | 0 | 是否启用纹理坐标映射变换 |
| mapping | matrix | 零矩阵 | 纹理坐标变换矩阵 |
| offset | float | 0.5 | 相邻行砖块的水平偏移量（0.5 表示半砖偏移） |
| offset_frequency | int | 2 | 偏移频率，每隔多少行应用偏移 |
| squash | float | 1.0 | 砖块挤压系数，用于改变特定行的砖块宽度 |
| squash_frequency | int | 1 | 挤压频率，每隔多少行应用挤压 |
| Vector | point | P | 输入纹理坐标 |
| Color1 | color | 0.2 | 砖块主颜色 |
| Color2 | color | 0.8 | 砖块次颜色（通过 Bias 混合） |
| Mortar | color | 0.0 | 灰缝颜色 |
| Scale | float | 5.0 | 纹理整体缩放 |
| MortarSize | float | 0.02 | 灰缝宽度 |
| MortarSmooth | float | 0.0 | 灰缝边缘平滑度（0 为硬边） |
| Bias | float | 0.0 | 砖块 Color1 与 Color2 混合偏移 |
| BrickWidth | float | 0.5 | 砖块宽度 |
| RowHeight | float | 0.25 | 砖块行高 |

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| Fac | float | 灰缝因子（0.0 = 砖块区域，1.0 = 灰缝区域） |
| Color | color | 最终混合颜色（砖块颜色与灰缝颜色的混合结果） |

## 实现逻辑

1. **坐标预处理**：对输入坐标应用 Scale 缩放，可选应用 mapping 变换矩阵。
2. **行号与砖号计算**：根据 RowHeight 计算行号 `rownum`，根据 BrickWidth 及偏移量计算砖号 `bricknum`。
3. **偏移与挤压**：根据 `offset_frequency` 和 `squash_frequency` 对特定行施加水平偏移和宽度挤压。
4. **灰缝判定**：计算采样点到砖块四边的最小距离 `min_dist`，若 `min_dist >= MortarSize` 则位于砖块内部（返回 0.0），否则处于灰缝区域。
5. **灰缝平滑**：当 `MortarSmooth > 0` 时，使用 `smoothstep` 对灰缝边缘进行平滑过渡。
6. **颜色着色**：通过 `brick_noise` 函数基于行号和砖号生成伪随机色调 `tint`，将 Color1 和 Color2 按 tint 混合得到砖块颜色，再与灰缝颜色按 Fac 混合输出最终颜色。

## 对应 SVM 节点

对应 `svm_brick` 相关实现（`src/kernel/svm/brick.h`）。
