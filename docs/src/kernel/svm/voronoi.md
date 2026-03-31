# voronoi.h - Voronoi 纹理 SVM 节点

## 概述
`voronoi.h` 实现了 Cycles 的 Voronoi 纹理节点(`NODE_TEX_VORONOI`)，这是一种基于 Voronoi 细胞分割的程序化纹理。代码基于 Inigo Quilez 的 Smooth Voronoi 算法，提供 1D 至 4D 四种维度，支持五种特征模式(F1/F2/Smooth F1/到边缘距离/N 球体半径)和四种距离度量(欧几里得/曼哈顿/切比雪夫/闵可夫斯基)，并支持分形叠加。

该文件是 SVM 中最大的单个节点实现之一，包含大量的维度特化代码和距离计算优化。

## 核心函数

### 数据结构
- **`VoronoiParams`** - 参数结构体：`scale`, `detail`, `roughness`, `lacunarity`, `smoothness`, `exponent`, `randomness`, `max_distance`, `normalize`, `feature`, `metric`
- **`VoronoiOutput`** - 输出结构体：`distance`(距离), `color`(颜色), `position`(位置)

### 距离计算函数
- **`voronoi_distance(a, b)`** - 1D 绝对值距离
- **`voronoi_distance<T>(a, b, params)`** - 通用模板距离，根据 `params.metric` 分发：
  - 欧几里得: `distance(a, b)`
  - 曼哈顿: `reduce_add(fabs(a - b))`
  - 切比雪夫: `reduce_max(fabs(a - b))`
  - 闵可夫斯基: `pow(reduce_add(pow(fabs(a-b), exp)), 1/exp)`
- **`voronoi_distance_bound<T>(a, b, params)`** - 低成本距离估计，用于比较排序（欧几里得模式使用 `len_squared` 避免开根号）

### 1D/2D/3D/4D Voronoi 核心（每维度一组）
- **`voronoi_f1(params, coord)`** - 最近特征点(F1)：遍历邻域格子，找到距离最近的随机特征点
- **`voronoi_smooth_f1(params, coord)`** - 平滑 F1：使用指数加权平滑多个特征点的贡献
- **`voronoi_f2(params, coord)`** - 第二近特征点(F2)：同时追踪最近和次近两个点
- **`voronoi_distance_to_edge(params, coord)`** - 到 Voronoi 边缘的距离：先找最近点，再扫描邻域计算到中垂面的距离
- **`voronoi_n_sphere_radius(params, coord)`** - N 球体半径：计算以特征点为中心、不超出 Voronoi 格子的最大球半径

### 分形 Voronoi
- **`fractal_voronoi_x_fx(params, coord)`** - 分形 Voronoi F1/Smooth F1/F2：多八度叠加 Voronoi 噪声
- **`fractal_voronoi_distance_to_edge(params, coord)`** - 分形 Voronoi 边缘距离

### SVM 节点入口
- **`svm_voronoi_output(stack_offsets, stack, ...)`** - 将计算结果写入 SVM 栈
- **`svm_node_tex_voronoi<node_feature_mask>(kg, stack, ...)`** - SVM 节点主函数，解包参数并按维度/特征分发计算

## 依赖关系
- **内部头文件**:
  - `kernel/svm/util.h` - 栈操作和节点读取
  - `util/hash.h` - 哈希函数，用于生成随机特征点位置和颜色
- **被引用**:
  - `kernel/svm/svm.h` - 主解释器通过 `NODE_TEX_VORONOI` case 调用

## 实现细节

### 特征点随机化
每个格子中的特征点位置通过哈希函数从格子坐标生成随机偏移，再用 `randomness` 参数在格子中心和随机位置之间插值：
```
point_position = cell + mix(0.5, hash(cell), randomness)
```
`randomness = 0` 时特征点在格子中心（规则网格），`= 1` 时完全随机。

### Smooth F1 平滑算法
使用指数加权求和而非简单取最小值：
```
h = smoothstep(0, 1, 0.5 + 0.5 * (d - minDist) / smoothness)
```
`smoothness` 参数控制平滑程度，生成类似元球(Metaball)的平滑过渡效果。

### 到边缘距离(Distance to Edge)优化
原始算法需要扫描 -2..2 的 5x5 邻域窗口，但代码引用了 Shadertoy 上的优化方案，将扫描范围缩减到 -1..1 的 3x3 窗口以提升性能。算法分两步：
1. 找到最近特征点
2. 在邻域中扫描其他特征点，计算当前点到两个特征点中垂面的距离

### 分形 Voronoi 叠加
分形版本使用与分形噪声类似的多八度叠加策略，每个八度的坐标乘以 `lacunarity`，振幅乘以 `roughness`。对于距离和颜色输出分别累积，位置输出取最大权重八度的值。

### 归一化支持
当 `normalize = true` 时，距离输出除以 `max_distance`（理论最大距离），将输出映射到 [0, 1] 范围。`max_distance` 根据距离度量和维度预计算。

## 关联文件
- `kernel/svm/types.h` - `NodeVoronoiFeature`, `NodeVoronoiDistanceMetric` 枚举
- `kernel/svm/svm.h` - SVM 主解释器
- `util/hash.h` - 哈希函数生成随机特征点
