# bvh2.h / bvh2.cpp - 二叉 BVH 构建、打包与重拟合实现

## 概述

本文件实现了 Cycles 渲染器中标准的二叉层次包围体(BVH2),每个内部节点恰好有两个子节点。`BVH2` 类继承自 `BVH` 基类,负责完整的构建流程:调用 `BVHBuild` 生成 BVH 树,然后将树结构打包(pack)为紧凑的 `int4` 数组供 GPU/CPU 渲染内核遍历。同时提供重拟合(refit)功能,在几何体变形时无需重建树结构,只需更新包围盒即可。

## 类与结构体

### BVHStackEntry
- **功能**: 辅助结构体,用于在打包节点时维护遍历栈。
- **关键成员**:
  - `node` (`const BVHNode *`) — 指向 BVH 树节点
  - `idx` (`int`) — 节点在打包数组中的索引
- **关键方法**:
  - `encodeIdx()` — 对叶节点索引进行按位取反编码,以区分叶节点和内部节点

### BVH2
- **继承**: `BVH`（定义于 `bvh/bvh.h`）
- **功能**: 二叉 BVH 的构建、打包、重拟合及实例合并的完整实现。
- **关键成员**:
  - `pack` (`PackedBVH`) — 打包后的 BVH 数据,包含节点数组、叶节点数组和图元索引
- **关键方法**:
  - `build(Progress &progress, Stats *stats)` — 完整构建流程:建树->打包图元->打包节点
  - `refit(Progress &progress)` — 重拟合:更新图元信息->更新节点包围盒
  - `widen_children_nodes(unique_ptr<BVHNode> &&root)` — 虚方法,允许子类扩展节点宽度（BVH2 中直接返回不做修改）
  - `pack_nodes(const BVHNode *root)` — 将树结构打包为 `int4` 数组
  - `pack_leaf(...)` — 打包叶节点
  - `pack_inner(...)` — 打包内部节点（自动选择对齐/非轴对齐）
  - `pack_aligned_node(...)` — 打包轴对齐内部节点（4 个 `int4`）
  - `pack_unaligned_node(...)` — 打包非轴对齐内部节点（7 个 `int4`）
  - `refit_nodes()` / `refit_node(...)` — 递归重拟合节点包围盒
  - `refit_primitives(...)` — 重拟合图元范围的包围盒
  - `pack_primitives()` — 打包图元可见性信息
  - `pack_instances(...)` — 合并实例化对象的子 BVH

### 预定义常量
- `BVH_NODE_SIZE = 4` — 轴对齐内部节点占用的 `int4` 数量
- `BVH_NODE_LEAF_SIZE = 1` — 叶节点占用的 `int4` 数量
- `BVH_UNALIGNED_NODE_SIZE = 7` — 非轴对齐内部节点占用的 `int4` 数量

## 核心函数

### `BVH2::build()`
1. 创建 `BVHBuild` 实例并调用 `run()` 构建 BVH 树
2. 调用 `widen_children_nodes()` 允许子类调整树结构
3. 调用 `pack_primitives()` 填充图元可见性数组
4. 调用 `pack_nodes()` 将树打包为设备可用格式

### `BVH2::pack_nodes()`
使用显式栈(非递归)遍历 BVH 树:
1. 统计总节点数和叶节点数,计算所需存储空间
2. 顶层 BVH 需先调用 `pack_instances()` 合并子 BVH
3. 使用栈遍历树,为每个节点分配索引并打包

### `BVH2::pack_aligned_node()`
每个轴对齐内部节点打包为 4 个 `int4`:
- `data[0]`: 左右子节点可见性标志、左右子节点索引
- `data[1]`: 左右子节点 X 轴最小/最大值
- `data[2]`: 左右子节点 Y 轴最小/最大值
- `data[3]`: 左右子节点 Z 轴最小/最大值

### `BVH2::pack_unaligned_node()`
每个非轴对齐内部节点打包为 7 个 `int4`:
- `data[0]`: 可见性标志（含 `PATH_RAY_NODE_UNALIGNED` 标记）和子节点索引
- `data[1-3]`: 左子节点的 3x4 变换矩阵
- `data[4-6]`: 右子节点的 3x4 变换矩阵

### `BVH2::refit_primitives()`
遍历图元范围,根据类型重新计算包围盒:
- 三角形:从顶点数据重新计算,含运动模糊步
- 曲线:从控制点和半径重新计算,含运动模糊步
- 点云:从点位置和半径重新计算,含运动模糊步
- 对象实例:使用对象包围盒

### `BVH2::pack_instances()`
合并顶层 BVH 和各实例对象的子 BVH:
1. 调整顶层图元索引偏移
2. 遍历所有几何体的子 BVH,合并其节点和图元数据
3. 修正子 BVH 中的节点索引偏移
4. 使用 `geometry_map` 避免重复合并相同几何体

## 依赖关系

- **内部头文件**:
  - `bvh/bvh.h` — 基类 `BVH` 和 `PackedBVH`
  - `bvh/build.h` — `BVHBuild` 构建器
  - `bvh/node.h` — `BVHNode`、`InnerNode`、`LeafNode`
  - `bvh/unaligned.h` — `BVHUnaligned::compute_node_transform()`
  - `bvh/params.h` — `BVHParams`
  - `scene/mesh.h`、`scene/hair.h`、`scene/object.h`、`scene/pointcloud.h` — 场景几何体
  - `util/progress.h` — 进度报告
- **被引用**:
  - `bvh/bvh.cpp` — 工厂方法中创建 `BVH2` 实例
  - `scene/geometry_bvh.cpp` — 几何体 BVH 构建调用 `BVH2::build()`
  - `device/device.cpp` — 设备端 BVH 数据访问

## 实现细节 / 关键算法

### 节点索引编码
叶节点和内部节点共用索引空间但分开存储。叶节点索引通过按位取反(`~idx`)编码为负数,渲染内核通过符号位区分叶节点和内部节点。

### 对齐 vs 非轴对齐节点
`pack_inner()` 检查子节点的 `is_unaligned` 标志。非轴对齐节点在可见性标志中设置 `PATH_RAY_NODE_UNALIGNED` 位,渲染内核据此选择不同的遍历路径。非轴对齐节点存储完整的 3x4 变换矩阵而非 AABB。

### 实例合并
顶层 BVH 通过 `pack_instances()` 将各几何体的独立子 BVH 合并到全局数组中。使用 `unordered_map<Geometry *, int>` 记录已合并的几何体,避免相同几何体被多个对象实例引用时重复合并。

### 重拟合优化
`refit()` 不重建树结构,只自底向上更新包围盒。适用于动画中几何体变形但拓扑不变的情况(如角色蒙皮动画),比完整重建快得多。

## 关联文件

- `bvh/bvh.h` / `bvh/bvh.cpp` — BVH 基类和工厂方法
- `bvh/build.h` / `bvh/build.cpp` — BVH 树构建器
- `bvh/node.h` / `bvh/node.cpp` — BVH 节点类型
- `bvh/unaligned.h` / `bvh/unaligned.cpp` — 非轴对齐变换计算
- `bvh/params.h` — BVH 参数
