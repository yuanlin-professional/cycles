# disjoint_set.h - 并查集（不相交集合）数据结构

## 概述

`disjoint_set.h` 实现了经典的并查集（Union-Find）数据结构，使用按秩合并（union by rank）和路径压缩（path compression）两种优化策略。并查集用于高效管理元素的分组关系，支持近乎常数时间的查找和合并操作，在 Cycles 中可用于连通分量检测等场景。

## 类与结构体

### `class DisjointSet`

并查集实现类。

**成员变量（private）：**
- `array<size_t> parents` -- 父节点数组，`parents[i]` 存储元素 `i` 的父节点索引
- `array<size_t> ranks` -- 秩数组，`ranks[i]` 存储以元素 `i` 为根的子树的上界高度

**构造函数：**
- `DisjointSet(const size_t size)` -- 创建包含 `size` 个独立元素的并查集，每个元素初始为自己的父节点，秩为 0

**公有方法：**

- `size_t find(size_t x)` -- 查找元素 `x` 的根节点（集合代表元素）。采用两阶段路径压缩：第一阶段沿父指针找到根节点，第二阶段将路径上所有节点直接指向根节点，将后续查找的时间复杂度降至接近 O(1)

- `void join(const size_t x, const size_t y)` -- 合并包含元素 `x` 和 `y` 的两个集合。采用按秩合并策略：将秩较小的树挂载到秩较大的树下；若两树秩相等则任选一方并将其秩加一。若 `x` 和 `y` 已在同一集合中则不做任何操作

## 核心函数

该文件中所有功能均封装在 `DisjointSet` 类内部，无独立函数。

## 依赖关系

- **内部头文件**: `util/array.h`
- **标准库**: `<utility>`（用于 `std::swap`）
- **被引用**: 当前代码库中仅在 `CMakeLists.txt` 中注册，未发现直接 `#include` 引用（可能用于特定构建配置或未来功能扩展）

## 关联文件

- `util/array.h` -- 提供底层存储容器，避免使用 `std::vector` 的构造/析构开销
