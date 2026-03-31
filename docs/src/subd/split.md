# split.h / split.cpp - DiagSplit 自适应面片分割算法

## 概述

本文件实现了 DiagSplit（对角线分割）算法，这是一种并行、无裂缝的自适应曲面细分方法，用于微多边形渲染。DiagSplit 负责将原始面片递归分割为足够小的子面片，并为每条边确定曲面细分因子（T 值），供后续的 EdgeDice 切割阶段使用。该算法的核心思想来自同名论文"DiagSplit: Parallel, Crack-free, Adaptive Tessellation for Micro-polygon Rendering"，支持在任意点对面片求值。

## 类与结构体

### DiagSplit

- **功能**: 自适应面片分割器，将网格面片递归分割为可直接切割的子面片集合
- **关键成员**:
  - `params` — 细分参数（`SubdParams`）
  - `subpatches` — 分割产出的子面片数组（`vector<SubPatch>`）
  - `owned_verts` — 顶点所有权标记数组（`vector<bool>`）
  - `edges` — 全局边集合（`unordered_set<SubEdge>`），通过哈希实现边的共享
  - `num_verts` — 当前分配的顶点总数
  - `num_triangles` — 当前分配的三角形总数
- **关键方法**:
  - `split_patches(patches, patches_byte_stride)` — 主入口，分割所有面片
  - `split_quad(SubPatch &&sub)` — 递归分割四边形子面片
  - `split_triangle(SubPatch &&sub)` — 递归分割三角形子面片
  - `split_quad_into_triangles(SubPatch &&sub)` — 将四边形拆分为两个三角形
  - `split_quad(face, face_index, patch)` — 从原始四边形面创建初始子面片
  - `split_ngon(face, face_index, patches, stride)` — 将 N-gon 面拆分为 N 个四边形子面片
  - `split_edge(...)` — 分割一条边，分配中间顶点和子边
  - `alloc_verts(num)` — 分配顶点索引
  - `alloc_edge(...)` — 分配或查找共享边
  - `alloc_subpatch(SubPatch&&)` — 最终确认子面片，分配内部网格顶点和三角形
  - `T(patch, uv_start, uv_end, depth, recursive_resolve)` — 计算边的细分因子
  - `limit_edge_factor(...)` — 限制边因子不超过最大级别
  - `assign_edge_factor(...)` — 为边设置细分因子并分配边顶点
  - `resolve_edge_factors(sub)` — 解析子面片所有边的细分因子
  - `to_world(patch, uv)` — 将面片参数坐标转换为世界空间坐标
  - `get_num_subpatches()` / `get_subpatch(i)` — 访问分割结果
  - `get_num_verts()` / `get_num_triangles()` — 获取分配的顶点和三角形总数

## 枚举与常量

使用 `subpatch.h` 中定义的常量：
- `DSPLIT_NON_UNIFORM = -1` — 标记边需要进一步分割（非均匀）
- `DSPLIT_MAX_DEPTH = 16` — 最大递归分割深度
- `DSPLIT_MAX_SEGMENTS = 8` — 单条边最大段数

## 核心函数

### DiagSplit::split_patches()
- **签名**: `void split_patches(const Patch *patches, const size_t patches_byte_stride)`
- **功能**: 主入口函数。遍历网格所有细分面，对四边形面调用 `split_quad`，对 N-gon 面调用 `split_ngon`。初始化顶点计数为基础网格顶点数。

### DiagSplit::T()
- **签名**: `std::pair<int, float> T(const Patch *patch, float2 uv_start, float2 uv_end, int depth, bool recursive_resolve)`
- **功能**: 计算边的曲面细分因子。沿边均匀采样 `test_steps` 个点，计算总弧长和最大分段长度，据此得到最小和最大细分因子。若两者之差超过 `split_threshold`，标记为 `DSPLIT_NON_UNIFORM`（需进一步分割）；在递归解析模式下，会将边一分为二递归计算。支持相机视角自适应：通过 `world_to_raster_size` 将世界空间长度转换为像素宽度。返回值包含细分因子和世界空间长度。

### DiagSplit::split_quad(SubPatch&&)
- **签名**: `void split_quad(SubPatch &&sub)`
- **功能**: 递归分割四边形子面片。首先解析边因子，然后根据以下条件决定是否分割：
  - 边标记为必须分割（`must_split()`）
  - 最小段数 >= 2 且对侧边超过 `DSPLIT_MAX_SEGMENTS` 且对侧比率超过 1.5

  若需要在两个方向都分割，选择较长的轴。若无法均匀分割（某侧仅1段），退化为三角形分割。分割时将子面片一分为二（沿选定轴的中点），创建新的共享边，递归处理两个子面片。

### DiagSplit::split_triangle(SubPatch&&)
- **签名**: `void split_triangle(SubPatch &&sub)`
- **功能**: 递归分割三角形子面片。选择需要分割的最长边，沿该边中点分割为两个三角形子面片，创建从中点到对角的新边，递归处理。

### DiagSplit::split_ngon()
- **签名**: `void split_ngon(const Mesh::SubdFace &face, int face_index, const Patch *patches, size_t patches_byte_stride)`
- **功能**: 将 N-gon 面分解为 N 个四边形子面片。每个四边形由角顶点、两条相邻边的中点和面中心点组成。先为 N 条边各分配中间顶点，然后为每个角创建子面片并递归分割。

### DiagSplit::split_edge()
- **签名**: `float2 split_edge(const Patch*, SubPatch::Edge*, SubPatch::Edge*, SubPatch::Edge*, float2, float2)`
- **功能**: 分割一条边。若边标记为 `must_split()`，在中点分配新顶点并创建两条子边；若 T >= 2，则在现有边顶点数组中取中间位置作为分割点。返回分割点的 UV 坐标。

## 依赖关系

- **内部头文件**: `scene/mesh.h`, `subd/dice.h`, `subd/subpatch.h`, `util/set.h`, `util/types.h`, `util/vector.h`, `scene/camera.h`, `subd/patch.h`, `util/algorithm.h`, `util/math.h`
- **外部库**: 无
- **被引用**: `subd/dice.cpp`, `scene/mesh_subdivision.cpp`, `scene/mesh.cpp`, `scene/geometry.cpp`

## 实现细节 / 关键算法

1. **边共享机制**: 使用 `unordered_set<SubEdge>` 存储所有边，以 (start_vert, end_vert) 为键进行哈希查找。相邻子面片共享同一条 SubEdge 对象，确保两侧看到相同的细分因子 T，从而保证无裂缝。
2. **顶点所有权**: 为避免在并行切割阶段多个子面片重复设置同一顶点的属性，使用 `own_vertex` 和 `own_edge` 标记来确定哪个子面片负责设置哪些顶点。
3. **自适应细分因子计算**: T 函数综合考虑边的弧长（Lsum）和最大分段长度（Lmax）。当 Lmax 导致的细分因子远大于 Lsum 导致的因子时（差异超过阈值），认为边不均匀，需要进一步分割。这避免了在弯曲变化剧烈的区域使用过少的三角形。
4. **分割方向选择**: 对于四边形，当两个方向都需要分割时，选择总边长较长的方向。引入微小偏差因子（1.00012345）以确保正方形面片在不同平台上有一致的分割方向。
5. **N-gon 到四边形**: N-gon 通过引入面中心点和边中点被分解为 N 个四边形，每个四边形覆盖原始面的一个角。这使得后续处理统一为四边形分割。
6. **深度限制**: 在接近 `DSPLIT_MAX_DEPTH - 3` 时，强制将 `DSPLIT_NON_UNIFORM` 的边设为 `DSPLIT_MAX_SEGMENTS`，确保递归终止。减3是为三角形面片的3条边各留一次分割机会。

## 关联文件

- `subd/dice.h` / `subd/dice.cpp` — EdgeDice 消费 DiagSplit 的输出
- `subd/subpatch.h` — SubPatch 和 SubEdge 数据结构
- `subd/patch.h` / `subd/patch.cpp` — Patch::eval 面片求值
- `scene/mesh_subdivision.cpp` — 调用 split_patches 启动分割
- `scene/mesh.h` — SubdFace 等数据结构
- `scene/camera.h` — 视角自适应计算
