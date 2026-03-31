# node_voronoi.h - Voronoi 基础函数库头文件

## 概述

该头文件是开放着色语言(OSL)着色器系统的 Voronoi 纹理核心函数库。定义了 Voronoi 纹理所需的数据结构、距离度量函数，以及 1D/2D/3D/4D 四个维度下的 F1、F2、平滑 F1、到边缘距离和 N 球半径五种 Voronoi 特征的完整实现。

## 文件依赖

```
node_voronoi.h
  ├── node_hash.h          (哈希函数库)
  ├── stdcycles.h           (Cycles 标准库)
  ├── vector2.h             (vector2 类型)
  └── vector4.h             (vector4 类型)
```

## 数据结构

### VoronoiParams

```osl
struct VoronoiParams {
    float scale;          // 纹理缩放
    float detail;         // 分形细节层级
    float roughness;      // 分形粗糙度
    float lacunarity;     // 分形缺陷度
    float smoothness;     // 平滑 F1 平滑度
    float exponent;       // 闵可夫斯基指数
    float randomness;     // 点分布随机性
    float max_distance;   // 最大距离（用于归一化）
    int normalize;        // 是否归一化
    string feature;       // 特征类型
    string metric;        // 距离度量
};
```

### VoronoiOutput

```osl
struct VoronoiOutput {
    float Distance;       // 距离值
    color Color;          // 随机颜色
    vector4 Position;     // 特征点位置
};
```

## 距离度量函数

`voronoi_distance` 系列函数支持四种距离度量（以 2D 为例）：

| 度量类型 | 公式 |
|----------|------|
| euclidean | `length(a - b)` |
| manhattan | `|ax-bx| + |ay-by|` |
| chebychev | `max(|ax-bx|, |ay-by|)` |
| minkowski | `(|ax-bx|^e + |ay-by|^e)^(1/e)` |

所有度量函数对 float、vector2、vector3、vector4 类型均有实现。1D 版本始终使用绝对值距离。

## Voronoi 特征函数

每种特征在 1D/2D/3D/4D 四个维度下均有独立实现。

### voronoi_f1 - 最近点距离

遍历当前单元及其邻域（3x3 / 3x3x3 / 3x3x3x3），找到距离最近的 Voronoi 特征点。输出最近距离、该点的哈希颜色和位置。特征点位置由 `hash_intN_to_vectorN * randomness` 在单元内偏移。

### voronoi_smooth_f1 - 平滑最近点距离

在扩展的邻域（5x5 等）中遍历，使用 `smoothstep` 和混合因子实现多个 Voronoi 点之间的平滑过渡。修正因子 `smoothness * h * (1 - h)` 确保插值连续性。基于 Inigo Quilez 的平滑 Voronoi 算法。

### voronoi_f2 - 次近点距离

同时追踪最近和次近两个点的距离，输出次近点的距离、颜色和位置。用于创建更复杂的 Voronoi 图案。

### voronoi_distance_to_edge - 到边缘距离

两遍扫描算法：第一遍找到最近 Voronoi 点，第二遍计算到最近点与其他点之间边缘的垂直距离。基于 Inigo Quilez 的 Voronoi 边缘距离算法。

### voronoi_n_sphere_radius - N 球半径

找到最近点及该点的最近邻点，返回两点间距离的一半。表示以最近 Voronoi 点为圆心、不与邻域重叠的最大球体半径。

## 被引用文件

- `node_fractal_voronoi.h` - 分形 Voronoi 封装
- `node_voronoi_texture.osl` - Voronoi 纹理着色器（间接引用）
