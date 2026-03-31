# osd.h / osd.cpp - OpenSubdiv 集成封装

## 概述

本文件封装了 Cycles 对 OpenSubdiv 库的集成，提供了 Catmull-Clark 细分曲面的拓扑构建、自适应细化、面片表生成和求值功能。全部代码受 `WITH_OPENSUBDIV` 条件编译宏保护。该模块是实现高质量平滑细分（相对于线性细分）的关键基础设施，通过特化 OpenSubdiv 的 `TopologyRefinerFactory` 模板将 Cycles 的网格数据适配到 OpenSubdiv 的拓扑描述中。

## 类与结构体

### OsdValue<T>

- **功能**: OpenSubdiv 接口所需的值包装器模板，实现 Clear 和 AddWithWeight 接口
- **关键方法**:
  - `Clear()` — 将值清零
  - `AddWithWeight(src, weight)` — 加权累加 `value += src.value * weight`

### OsdMesh

- **功能**: Cycles 网格到 OpenSubdiv 拓扑的适配器封装
- **关键成员**:
  - `mesh` — Cycles 网格引用
  - `merged_fvars` — 合并后的 Face-Varying 属性列表（`vector<MergedFVar>`）
- **关键方法**:
  - `sdc_options()` — 构建 OpenSubdiv 的 Sdc::Options，映射 Cycles 的 FVar 线性插值模式和边界插值模式
  - `use_smooth_fvar(attr)` — 判断某个属性是否需要平滑 FVar 插值（UV 贴图或标记为 ATTR_SUBDIVIDE_SMOOTH_FVAR 的角点属性）
  - `use_smooth_fvar()` — 判断网格中是否有任何属性需要平滑 FVar 插值

### OsdMesh::MergedFVar

- **功能**: 存储合并后的 Face-Varying 属性数据
- **关键成员**:
  - `attr` — 原始属性引用
  - `channel` — 对应的 OpenSubdiv FVar 通道编号
  - `values` — 合并后的属性值缓冲区（`vector<char>`，泛型存储）

### OsdData

- **功能**: OpenSubdiv 细化器和面片数据结构的持有者
- **关键成员**:
  - `refiner` — 拓扑细化器（`unique_ptr<Far::TopologyRefiner>`）
  - `patch_table` — 面片表（`unique_ptr<Far::PatchTable>`）
  - `patch_map` — 面片映射，用于从参数坐标查找面片句柄（`unique_ptr<Far::PatchMap>`）
  - `refined_verts` — 细化后的顶点位置数据（`vector<OsdValue<float3>>`）
- **关键方法**:
  - `build(OsdMesh &osd_mesh)` — 构建完整的细化管线：创建拓扑细化器 -> 自适应细化 -> 生成面片表 -> 插值顶点 -> 创建面片映射

### OsdPatch

- **继承**: `Patch`（final）
- **功能**: 基于 OpenSubdiv 面片表的面片求值实现
- **关键成员**: `osd_data` — OsdData 引用
- **关键方法**:
  - `eval(P, dPdu, dPdv, N, u, v)` — 在面片上求值位置、偏导数和法线。使用 PatchMap::FindPatch 定位面片，通过 EvaluateBasis 计算基函数权重，加权求和得到位置和导数。

## 枚举与常量

无独立枚举。使用 OpenSubdiv 命名空间别名：
- `Far` = `OpenSubdiv::Far`
- `Sdc` = `OpenSubdiv::Sdc`

## 核心函数

### TopologyRefinerFactory<OsdMesh> 特化

osd.cpp 中特化了 OpenSubdiv 的 `TopologyRefinerFactory<OsdMesh>` 模板的以下方法：

#### resizeComponentTopology()
- **功能**: 设置基础网格的顶点数和面数，以及每个面的顶点数

#### assignComponentTopology()
- **功能**: 填充每个面的顶点索引（从 Mesh 的 subd_face_corners 数据中读取）

#### assignComponentTags()
- **功能**: 设置边和顶点的折痕权重（crease）。使用历史上 Pixar 采用的最大折痕权重 10.0 作为缩放系数。对于只有两条邻接边的顶点，额外累加较小的边折痕权重。

#### assignFaceVaryingTopology()
- **功能**: 为需要平滑 FVar 插值的属性创建 FVar 通道。使用 `merge_smooth_fvar<T>()` 将同一顶点上值相同的角点合并，生成紧凑的 FVar 拓扑。

#### reportInvalidTopology()
- **功能**: 记录无效拓扑的警告日志

### merge_smooth_fvar<T>()
- **签名**: `static void merge_smooth_fvar(const Mesh&, const Attribute&, OsdMesh::MergedFVar&, vector<int>&, vector<int>&)`
- **功能**: 合并同一顶点上值相同的 Face-Varying 角点数据。使用链表结构跟踪每个顶点的多个不同值，相同值复用索引，不同值追加到末尾。

### OsdData::build()
- **签名**: `void build(OsdMesh &osd_mesh)`
- **功能**: 完整的 OpenSubdiv 构建管线。使用 Catmull-Clark 方案、自适应细化（最大隔离级别 3）、Gregory 基端帽（ENDCAP_GREGORY_BASIS）、无限锐边面片支持。

### OsdPatch::eval()
- **签名**: `void eval(float3 *P, float3 *dPdu, float3 *dPdv, float3 *N, const float u, const float v) const`
- **功能**: 使用 PatchMap 定位参数坐标所在的面片，通过 EvaluateBasis 获取基函数权重（最多 20 个控制点），加权求和得到位置、U/V 方向偏导数，并通过叉积计算法线。

## 依赖关系

- **内部头文件**: `subd/patch.h`, `util/unique_ptr.h`, `util/vector.h`, `scene/attribute.h`, `scene/mesh.h`, `util/log.h`
- **外部库**: OpenSubdiv（`far/patchMap.h`, `far/patchTableFactory.h`, `far/primvarRefiner.h`, `far/topologyRefinerFactory.h`）
- **被引用**: `subd/interpolation.h`, `scene/mesh_subdivision.cpp`

## 实现细节 / 关键算法

1. **FVar 合并算法**: 对于 UV 等 Face-Varying 属性，同一顶点可能有多个不同值（UV 接缝处）。merge_smooth_fvar 使用基于数组的链表结构：每个顶点有一个链表头指向其第一个值，后续不同的值追加到数组末尾并链接起来。相同值直接复用索引，实现了高效的拓扑压缩。
2. **自适应细化**: 使用 `RefineAdaptive` 而非均匀细化，仅在需要的区域增加细分级别（如折痕边、非正则顶点附近），大幅减少面片数量。
3. **Sdc::Options 映射**: 将 Cycles 的6种 FVar 线性插值模式和3种边界插值模式一一映射到 OpenSubdiv 对应的选项。
4. **折痕权重处理**: 边折痕直接设置到对应边上。顶点折痕除了直接设置外，对于仅有两条邻接边的"角落"顶点，额外累加两条边中较小的折痕权重，以确保角落处的锐度正确。

## 关联文件

- `subd/patch.h` / `subd/patch.cpp` — Patch 基类定义
- `subd/interpolation.h` / `subd/interpolation.cpp` — 属性插值，依赖 OsdData 进行平滑细化
- `scene/mesh_subdivision.cpp` — 网格细分主流程，调用 OsdData::build
- `scene/mesh.h` — 网格数据结构
