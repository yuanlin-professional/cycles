# dice.h / dice.cpp - 细分曲面的边缘切割（Dicing）实现

## 概述

本文件实现了基于 DX11 风格的 EdgeDice 边缘切割算法，用于将细分曲面子面片（SubPatch）离散化为三角形网格。该算法为每条边使用独立的曲面细分因子，以实现无裂缝（watertight）的曲面细分，并通过子面片重映射配合 DiagSplit 算法工作。算法细节参考了 DiagSplit 论文及 ARB_tessellation_shader OpenGL 扩展规范（Section 2.X.2）。

## 类与结构体

### SubdParams

- **功能**: 存储细分曲面参数配置
- **关键成员**:
  - `mesh` — 目标网格指针（`Mesh*`）
  - `ptex` — 是否启用 PTex 纹理坐标
  - `test_steps` — 边缘细分因子计算时的采样步数，默认 3
  - `split_threshold` — 触发非均匀分割的阈值，默认 1
  - `dicing_rate` — 切割率，控制三角形密度，默认 1.0
  - `max_level` — 最大细分层级，默认 12
  - `camera` — 相机指针，用于视角自适应切割
  - `objecttoworld` — 物体到世界空间的变换矩阵

### EdgeDice

- **功能**: 核心边缘切割类，负责将 DiagSplit 产出的子面片离散化为网格三角形
- **关键成员**:
  - `params` — 细分参数（`SubdParams`）
  - `interpolation` — 属性插值器引用（`SubdAttributeInterpolation&`）
  - `mesh_triangles` / `mesh_shader` / `mesh_smooth` — 网格三角形索引、着色器及平滑标记数据指针
  - `mesh_P` / `mesh_N` — 顶点位置和法线数据指针
  - `mesh_ptex_face_id` / `mesh_ptex_uv` — PTex 面ID和UV坐标数据指针
- **关键方法**:
  - `dice(const DiagSplit &split)` — 主入口，对所有子面片执行并行切割
  - `tri_dice(const SubPatch &sub)` — 三角形子面片切割
  - `quad_dice(const SubPatch &sub)` — 四边形子面片切割
  - `set_vertex(...)` — 设置顶点位置、法线及属性插值
  - `set_triangle(...)` — 设置三角形索引、着色器属性及属性插值
  - `add_grid_triangles_and_stitch(...)` — 创建内部网格并缝合到边缘
  - `add_triangle_strip(...)` — 当一个方向仅有1段时以三角形条带方式缝合两侧边
  - `eval_projected(...)` — 在面片上求值并可选投影到光栅空间
  - `scale_factor(...)` — 计算网格缩放因子（当前未启用）
  - `quad_area(...)` — 计算四边形面积（拆分为两个三角形）

## 枚举与常量

无独立枚举或常量定义（使用 `subpatch.h` 中的 `DSPLIT_*` 常量）。

## 核心函数

### EdgeDice::dice()
- **签名**: `void dice(const DiagSplit &split)`
- **功能**: 对 DiagSplit 产出的所有子面片执行两遍并行处理：第一遍计算边缘顶点坐标（保证确定性），第二遍计算内部顶点坐标并生成三角形。使用 TBB 的 `parallel_for` 以8个子面片为粒度并行执行。

### EdgeDice::quad_dice()
- **签名**: `void quad_dice(const SubPatch &sub)`
- **功能**: 四边形子面片切割。根据 Mu/Mv（各方向最大段数）决定策略：若某方向仅1段则退化为三角形条带缝合；否则创建内部网格并与四条边缘缝合。

### EdgeDice::tri_dice()
- **签名**: `void tri_dice(const SubPatch &sub)`
- **功能**: 三角形子面片切割。支持 M=1（单三角形）、M=2（特殊情况处理1~3条边的2段分割）及一般情况（内部三角形网格+边缘缝合）。

### EdgeDice::add_grid_triangles_and_stitch()
- **签名**: `void add_grid_triangles_and_stitch(const SubPatch &sub, const int Mu, const int Mv)`
- **功能**: 创建 (Mu-1)x(Mv-1) 的内部顶点网格，生成内部三角形对，然后将内部网格的四条边与子面片的外部边缘进行缝合。缝合时使用最短对角线策略来保持三角形形状合理。

### EdgeDice::add_triangle_strip()
- **签名**: `void add_triangle_strip(const SubPatch &sub, const int left_edge, const int right_edge)`
- **功能**: 当一个方向仅有1段时，从左侧边到右侧边以三角形条带方式缝合，同样使用最短对角线策略。

## 依赖关系

- **内部头文件**: `util/transform.h`, `util/types.h`, `subd/subpatch.h`, `scene/camera.h`, `scene/mesh.h`, `subd/interpolation.h`, `subd/patch.h`, `subd/split.h`
- **外部库**: `util/tbb.h`（Intel TBB 并行库）
- **被引用**: `subd/split.h`, `subd/split.cpp`, `scene/mesh.h`

## 实现细节 / 关键算法

1. **两遍并行切割**: 第一遍设置边缘顶点坐标，确保共享边的顶点由确定的子面片写入；第二遍生成内部网格和三角形。两遍分离是为了保证边缘顶点在缝合时已经可用。
2. **最短对角线缝合**: 在缝合内外网格边缘时，算法比较两个候选对角线的长度平方，选择较短的方向来划分三角形，以避免产生退化或过于狭长的三角形。
3. **三角形特殊情况**: 对于 M=2 的三角形面片，根据有几条边需要2段分割，分别采用不同的三角剖分方案（1条、2条或3条边各有专门的模式）。
4. **PTex 支持**: 若启用 ptex，会额外写入 PTex 面ID和UV坐标属性。

## 关联文件

- `subd/split.h` / `subd/split.cpp` — DiagSplit 分割算法，产出子面片供切割使用
- `subd/subpatch.h` — SubPatch 和 SubEdge 数据结构定义
- `subd/patch.h` / `subd/patch.cpp` — 面片求值（Patch::eval）
- `subd/interpolation.h` / `subd/interpolation.cpp` — 属性插值
- `scene/mesh.h` — 网格数据结构
- `scene/camera.h` — 相机，用于视角自适应切割率计算
