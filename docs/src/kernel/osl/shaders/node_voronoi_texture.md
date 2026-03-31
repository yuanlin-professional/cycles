# node_voronoi_texture.osl - Voronoi 纹理着色器

## 概述

该开放着色语言(OSL)着色器节点生成基于 Voronoi 细胞的程序化纹理。支持 1D/2D/3D/4D 四个维度，提供 F1、F2、平滑 F1、到边缘距离和 N 球半径五种特征模式，以及欧几里得、曼哈顿、切比雪夫和闵可夫斯基四种距离度量。支持分形细节叠加，是创建细胞、石头、裂纹等自然纹理的核心节点。

## 着色器签名

```osl
shader node_voronoi_texture(
    int use_mapping = 0,
    matrix mapping = matrix(0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0),
    string dimensions = "3D",
    string feature = "f1",
    string metric = "euclidean",
    int use_normalize = 0,
    point Vector = P,
    float WIn = 0.0,
    float Scale = 5.0,
    float Detail = 0.0,
    float Roughness = 0.5,
    float Lacunarity = 2.0,
    float Smoothness = 5.0,
    float Exponent = 1.0,
    float Randomness = 1.0,
    output float Distance = 0.0,
    output color Color = 0.0,
    output point Position = P,
    output float WOut = 0.0,
    output float Radius = 0.0)
```

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| use_mapping | int | 0 | 是否启用纹理坐标映射变换 |
| mapping | matrix | 零矩阵 | 纹理坐标变换矩阵 |
| dimensions | string | "3D" | 维度："1D"、"2D"、"3D"、"4D" |
| feature | string | "f1" | 特征模式（见下方说明） |
| metric | string | "euclidean" | 距离度量（见下方说明） |
| use_normalize | int | 0 | 是否归一化距离输出 |
| Vector | point | P | 输入纹理坐标 |
| WIn | float | 0.0 | 第四维坐标（用于 1D 和 4D） |
| Scale | float | 5.0 | 纹理缩放 |
| Detail | float | 0.0 | 分形细节层级（0-15） |
| Roughness | float | 0.5 | 分形粗糙度（0-1） |
| Lacunarity | float | 2.0 | 分形缺陷度 |
| Smoothness | float | 5.0 | 平滑 F1 特征的平滑度 |
| Exponent | float | 1.0 | 闵可夫斯基度量的指数 |
| Randomness | float | 1.0 | 点分布随机性（0 = 规则网格，1 = 完全随机） |

**feature 可选值：**
- `"f1"` - 到最近 Voronoi 点的距离
- `"f2"` - 到第二近 Voronoi 点的距离
- `"smooth_f1"` - F1 的平滑版本，使用 smoothstep 插值
- `"distance_to_edge"` - 到 Voronoi 细胞边缘的距离
- `"n_sphere_radius"` - 最近点与次近点之间的 N 球半径

**metric 可选值：**
- `"euclidean"` - 欧几里得距离（标准直线距离）
- `"manhattan"` - 曼哈顿距离（轴对齐距离之和）
- `"chebychev"` - 切比雪夫距离（各轴距离的最大值）
- `"minkowski"` - 闵可夫斯基距离（由 Exponent 参数控制）

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| Distance | float | 计算得到的距离值 |
| Color | color | 基于哈希的随机颜色 |
| Position | point | 最近 Voronoi 点的位置 |
| WOut | float | 最近 Voronoi 点的 W 坐标分量 |
| Radius | float | N 球半径值（仅 n_sphere_radius 特征有效） |

## 实现逻辑

1. **参数封装**：将所有参数打包到 `VoronoiParams` 结构体中，包括缩放、细节、平滑度等。
2. **坐标缩放**：输入坐标乘以 Scale。
3. **特征分发**：
   - **distance_to_edge**：调用 `fractal_voronoi_distance_to_edge`，遍历邻域单元格找到最近点后，计算到 Voronoi 边缘的垂直距离。
   - **n_sphere_radius**：调用 `voronoi_n_sphere_radius`，找到最近点及其最近邻点，返回半距离。
   - **f1/f2/smooth_f1**：调用 `fractal_voronoi_x_fx`，该函数内部根据 feature 类型选择 `voronoi_f1`、`voronoi_f2` 或 `voronoi_smooth_f1`。
4. **分形叠加**：当 Detail > 0 时，使用类似 fBM 的多层级叠加策略，每层级按 lacunarity 放大坐标、按 roughness 衰减振幅。
5. **归一化**：当 `use_normalize` 启用时，距离除以最大振幅和最大距离进行归一化。

## 对应 SVM 节点

对应 `svm_voronoi` 相关实现（`src/kernel/svm/voronoi.h`）。
