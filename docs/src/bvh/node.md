# node.h / node.cpp - BVH 树节点类型定义与统计/调试功能

## 概述

本文件定义了 Cycles 渲染器层次包围体(BVH)树的节点类型层次结构。包含抽象基类 `BVHNode` 及其两个具体子类 `InnerNode`（内部节点）和 `LeafNode`（叶节点）。这些类用于 BVH 构建阶段表示树的中间状态,构建完成后由 `BVH2::pack_nodes()` 打包为紧凑的 `int4` 数组供渲染内核使用。同时提供树的统计信息计算、表面积启发式(SAH)代价评估、可见性更新和图形化调试输出等功能。

## 类与结构体

### BVH_STAT（枚举）
- **功能**: 定义 BVH 树统计类型
- **枚举值**:
  - `BVH_STAT_NODE_COUNT` — 总节点数
  - `BVH_STAT_INNER_COUNT` — 内部节点数
  - `BVH_STAT_LEAF_COUNT` — 叶节点数
  - `BVH_STAT_TRIANGLE_COUNT` — 三角形总数
  - `BVH_STAT_CHILDNODE_COUNT` — 子节点总数
  - `BVH_STAT_ALIGNED_COUNT` / `BVH_STAT_UNALIGNED_COUNT` — 轴对齐/非轴对齐节点数
  - `BVH_STAT_ALIGNED_INNER_COUNT` / `BVH_STAT_UNALIGNED_INNER_COUNT` — 轴对齐/非轴对齐内部节点数
  - `BVH_STAT_ALIGNED_LEAF_COUNT` / `BVH_STAT_UNALIGNED_LEAF_COUNT` — 轴对齐/非轴对齐叶节点数
  - `BVH_STAT_DEPTH` — 树的最大深度

### BVHNode（抽象基类）
- **继承**: 无
- **功能**: 所有 BVH 节点的抽象基类,定义通用接口和属性。
- **关键成员**:
  - `bounds` (`BoundBox`) — 节点包围盒
  - `visibility` (`uint`) — 可见性标志位掩码
  - `is_unaligned` (`bool`) — 是否为非轴对齐节点
  - `aligned_space` (`unique_ptr<Transform>`) — 非轴对齐变换空间（仅非轴对齐节点有效）
  - `time_from` / `time_to` (`float`) — 运动模糊时间范围
- **关键方法**:
  - `is_leaf()` — 纯虚方法,判断是否为叶节点
  - `num_children()` — 纯虚方法,返回子节点数量
  - `get_child(int i)` — 纯虚方法,获取第 i 个子节点
  - `num_triangles()` — 虚方法,返回三角形数量(默认 0)
  - `print(int depth)` — 纯虚方法,打印节点信息
  - `set_aligned_space(const Transform &)` — 设置非轴对齐变换空间
  - `get_aligned_space()` — 获取对齐变换空间
  - `has_unaligned()` — 检查是否有任何子节点为非轴对齐
  - `getSubtreeSize(BVH_STAT stat)` — 递归统计子树的各种统计量
  - `computeSubtreeSAHCost(const BVHParams &p, float probability)` — 递归计算子树的 SAH 代价
  - `update_visibility()` — 自底向上更新可见性标志
  - `update_time()` — 自底向上更新时间范围
  - `dump_graph(const char *filename)` — 将树导出为 Graphviz DOT 格式

### InnerNode
- **继承**: `BVHNode`
- **功能**: BVH 内部节点,包含 2 到 `kNumMaxChildren`（8）个子节点。
- **关键成员**:
  - `kNumMaxChildren = 8` — 最大子节点数量（编译期常量）
  - `num_children_` (`int`) — 实际子节点数量
  - `children[kNumMaxChildren]` (`unique_ptr<BVHNode>[]`) — 子节点数组
- **关键方法**:
  - `InnerNode(const BoundBox &bounds, unique_ptr<BVHNode> &&child0, unique_ptr<BVHNode> &&child1)` — 二子节点构造,设置可见性为子节点的按位或
  - `InnerNode(const BoundBox &bounds)` — 空构造,用于构建器稍后填充子节点
  - `is_leaf()` — 返回 `false`
  - `num_children()` — 返回 `num_children_`
  - `get_child(int i)` — 返回 `children[i].get()`

### LeafNode
- **继承**: `BVHNode`
- **功能**: BVH 叶节点,存储图元范围索引。
- **关键成员**:
  - `lo` (`int`) — 图元范围起始索引
  - `hi` (`int`) — 图元范围结束索引（不含）
- **关键方法**:
  - `LeafNode(const BoundBox &bounds, uint visibility, int lo, int hi)` — 构造叶节点
  - `is_leaf()` — 返回 `true`
  - `num_children()` — 返回 `0`
  - `num_triangles()` — 返回 `hi - lo`

## 核心函数

### `BVHNode::getSubtreeSize()`
递归遍历子树,根据 `BVH_STAT` 类型统计不同指标。对 `BVH_STAT_DEPTH` 特殊处理:返回所有子树深度的最大值加 1。

### `BVHNode::computeSubtreeSAHCost()`
递归计算子树的表面积启发式(SAH)代价:
```
SAH = probability * cost(num_children, num_triangles)
    + sum(child_probability * child_SAH)
```
其中子节点概率通过包围盒面积比计算:`child.bounds.safe_area() / bounds.safe_area()`。

### `BVHNode::update_visibility()`
自底向上合并可见性标志。对内部节点,将两个子节点的可见性按位或后赋值给自身。

### `BVHNode::update_time()`
自底向上合并时间范围。取两个子节点时间范围的并集(`min(time_from)`, `max(time_to)`)。

### `BVHNode::dump_graph()`
将 BVH 树导出为 Graphviz DOT 文件:
- 叶节点用浅蓝色(`#ccccee`)填充
- 内部节点用浅绿色(`#cceecc`)填充
- 每个节点标注唯一 ID

## 依赖关系

- **内部头文件**:
  - `util/boundbox.h` — `BoundBox` 包围盒
  - `util/types.h` — 基本类型
  - `util/unique_ptr.h` — `unique_ptr`
  - `bvh/build.h` — `BVHBuild`（.cpp 中引用,用于 SAH 计算参数）
  - `bvh/bvh.h` — BVH 基类（.cpp 中引用）
- **被引用**:
  - `bvh/build.cpp` — 构建过程中创建 `InnerNode` 和 `LeafNode`
  - `bvh/bvh2.cpp` — 打包阶段遍历节点树
  - `bvh/split.h` / `bvh/split.cpp` — 分割后创建节点

## 实现细节 / 关键算法

### 内存所有权
`InnerNode` 通过 `unique_ptr<BVHNode>` 持有子节点的所有权。整棵树的根节点由 `BVHBuild::run()` 返回的 `unique_ptr<BVHNode>` 持有,析构时自动递归释放所有节点。

### 最大子节点数
`kNumMaxChildren = 8` 允许未来扩展到更宽的 BVH（如 BVH4、BVH8）。当前 BVH2 实现中 `num_children_` 始终为 2,`widen_children_nodes()` 虚方法为子类提供了扩展宽度的扩展点。

### 拷贝构造
`BVHNode` 的拷贝构造函数深拷贝 `aligned_space`（如果存在），确保非轴对齐变换的独立性。`LeafNode` 使用默认拷贝构造。

### 调试输出
`dump_graph()` 生成的 DOT 文件可用 Graphviz 工具可视化 BVH 树结构,用于调试和性能分析。

## 关联文件

- `bvh/build.h` / `bvh/build.cpp` — BVH 构建器,创建和操作节点树
- `bvh/bvh2.h` / `bvh/bvh2.cpp` — 将节点树打包为设备格式
- `bvh/params.h` — `BVHParams`,提供 SAH 代价计算参数
- `util/boundbox.h` — `BoundBox` 包围盒定义
