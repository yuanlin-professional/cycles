# subpatch.h - 子面片与子边数据结构定义

## 概述

本文件定义了细分曲面系统中的核心数据结构：SubEdge（子边）和 SubPatch（子面片）。这些结构在 DiagSplit 分割阶段创建，并在 EdgeDice 切割阶段被使用。SubEdge 存储边的曲面细分因子和顶点分配信息，SubPatch 存储子面片的参数域映射、边引用和网格拓扑计算功能。该文件还定义了分割算法使用的关键常量。

## 类与结构体

### SubEdge

- **功能**: 子边数据结构，存储边的细分信息和顶点索引
- **关键成员**:
  - `start_vert_index` / `end_vert_index` — 起始和结束顶点索引
  - `mid_vert_index` — 若边被分割，中间顶点索引（初始为 -1）
  - `T` — 边的曲面细分因子（段数），0 表示未计算，`DSPLIT_NON_UNIFORM` 表示需要进一步分割
  - `length` — 边的估计长度（世界空间），用于决定分割方向
  - `second_vert_index` — 沿边第二个顶点的索引起始位置（初始为 -1）
  - `depth` — 该边经过了多少次细分
- **关键方法**:
  - `get_vert_along_edge(n)` — 获取沿边第 n 个顶点的索引（0 = 起点，T = 终点，中间为 second_vert_index + n - 1）
  - `must_split()` — 返回 T == DSPLIT_NON_UNIFORM，即边是否必须继续分割
- **嵌套结构**:
  - `Hash` — 哈希函数对象，基于 (start, end) 顶点对，确保 (a,b) 和 (b,a) 相同
  - `Equal` — 相等比较函数对象，双向比较顶点索引对

### SubPatch

- **功能**: 子面片数据结构，表示分割后的一块参数域区域
- **关键成员**:
  - `patch` — 指向父面片的指针（`const Patch*`）
  - `face_index` — 原始网格面索引
  - `corner` — 对应的角索引（N-gon 分解时使用）
  - `shape` — 面片形状枚举：`TRIANGLE` 或 `QUAD`
  - `inner_grid_vert_offset` — 内部网格顶点的起始索引
  - `triangles_offset` — 三角形的起始索引
  - `uvs[4]` — 参数域中的角点 UV 坐标，逆时针排列，默认为单位正方形
  - `edges[4]` — 4条边的引用（三角形只用前3条）
- **关键方法**:
  - `calc_num_inner_verts()` — 计算内部网格顶点数。四边形为 (Mu-1)*(Mv-1)；三角形为 M*(M-1)/2（M <= 2 时为 0）
  - `calc_num_triangles()` — 计算三角形总数（内部三角形 + 边缘缝合三角形）
  - `map_uv(uv)` — 将子面片局部 UV 映射到父面片参数空间。三角形使用重心坐标插值，四边形使用双线性插值，结果钳制到 [0,1]
  - `get_vert_along_edge(edge, n)` — 获取沿指定边的第 n 个顶点
  - `get_vert_along_edge_reverse(edge, n)` — 反向获取沿边顶点
  - `get_inner_grid_vert_triangle(i, j)` — 获取三角形内部网格中 (i, j) 位置的顶点索引
  - `get_vert_along_grid_edge(edge, n)` — 获取内部网格沿某条边的第 n 个顶点索引

### SubPatch::Edge

- **功能**: 子面片中的边引用，包含方向和所有权信息
- **关键成员**:
  - `edge` — 指向共享 SubEdge 的指针
  - `reversed` — 该子面片看到的边方向是否与 SubEdge 存储方向相反
  - `own_vertex` — 是否负责设置起始顶点的属性
  - `own_edge` — 是否负责设置边上中间顶点的属性
- **关键方法**:
  - `start_vert_index()` / `end_vert_index()` — 根据 `reversed` 标记返回正确方向的顶点
  - `mid_vert_index()` — 返回中间顶点索引
  - `get_vert_along_edge(n_relative)` — 根据方向翻转后获取沿边顶点

## 枚举与常量

```cpp
enum {
  DSPLIT_NON_UNIFORM = -1,   // 边的 T 值标记：需要进一步分割
  DSPLIT_MAX_DEPTH = 16,     // 最大递归分割深度
  DSPLIT_MAX_SEGMENTS = 8,   // 单条边最大段数
};
```

## 核心函数

本文件为纯头文件，所有方法均为内联定义。

### SubPatch::calc_num_triangles()
- **功能**: 精确计算子面片将产生的三角形数量。对于四边形：若 Mu 或 Mv 为 1，退化为两条边的段数之和；否则为内部网格三角形 + 四边缘合三角形。对于三角形：M=1 时为 1 个三角形，M=2 时为特殊情况，一般情况为内部网格三角形 + 三边缝合三角形。

### SubPatch::map_uv()
- **签名**: `float2 map_uv(float2 uv) const`
- **功能**: 从子面片的局部参数空间 [0,1]x[0,1] 映射到父面片的参数空间。三角形使用 `(1-u-v)*uvs[0] + u*uvs[1] + v*uvs[2]`；四边形使用双线性插值 `lerp(lerp(uvs[0], uvs[3], v), lerp(uvs[1], uvs[2], v), u)`。结果被钳制到 [0,1] 范围内以避免数值越界。

### SubPatch::get_vert_along_grid_edge()
- **签名**: `int get_vert_along_grid_edge(const int edge, const int n) const`
- **功能**: 根据面片形状（三角形或四边形）和边编号，计算内部网格中沿该边方向第 n 个内部顶点的索引。三角形使用 `get_inner_grid_vert_triangle` 的对角线/行/列索引；四边形使用行列偏移计算。

## 依赖关系

- **内部头文件**: `util/hash.h`, `util/math_float2.h`
- **外部库**: 无
- **被引用**: `subd/dice.h`, `subd/split.h`, `subd/split.cpp`

## 实现细节 / 关键算法

1. **边共享与方向**: SubEdge 始终按 (start < end) 的规范化方向存储，SubPatch::Edge 通过 `reversed` 标记记录子面片实际使用的方向。这确保两个相邻子面片通过同一 SubEdge 对象看到相同的 T 值和顶点分配。
2. **三角形内部网格索引**: 三角形的内部网格按行存储，第 j 行有 j+1 个顶点（三角形金字塔排列），通过 `j*(j+1)/2 + i` 公式计算线性偏移。
3. **四边形内部网格索引**: 四边形的内部网格为 (Mu-1) x (Mv-1) 的规则网格，按行优先存储，通过 `i + j*(Mu-1)` 计算偏移。
4. **顶点所有权机制**: `own_vertex` 和 `own_edge` 确保在并行切割时每个顶点/边仅由一个子面片负责写入属性，避免竞态条件。

## 关联文件

- `subd/dice.h` / `subd/dice.cpp` — 消费 SubPatch 进行切割
- `subd/split.h` / `subd/split.cpp` — 创建和分割 SubPatch
- `subd/patch.h` — Patch 基类，SubPatch 通过 `patch` 指针引用
