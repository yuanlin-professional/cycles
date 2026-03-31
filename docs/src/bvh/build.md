# build.h / build.cpp - 层次包围体(BVH)树构建器

## 概述

本文件实现了 Cycles 渲染器的 BVH 树构建核心逻辑。`BVHBuild` 类负责从场景对象(三角形网格、毛发曲线、点云)收集图元引用,然后使用表面积启发式(SAH)算法递归构建二叉 BVH 树。支持两种分割策略:对象分箱(Object Binning)和空间分割(Spatial Split),并支持多线程并行构建和非轴对齐节点优化。构建完成后通过树旋转优化提升遍历性能。

## 类与结构体

### BVHBuild
- **继承**: 无
- **功能**: BVH 树的完整构建流程,包括图元引用收集、递归节点构建、多线程任务分发、进度报告和树旋转优化。
- **关键成员**:
  - `objects` — 场景对象列表
  - `references` — 图元引用数组(构建输入）
  - `num_original_references` — 原始引用数量（空间分割可能增加引用）
  - `prim_type` / `prim_index` / `prim_object` / `prim_time` — 输出图元索引数组（引用方式，直接写入 `PackedBVH`）
  - `params` — BVH 构建参数(`BVHParams`)
  - `progress` — 进度报告器
  - `spatial_min_overlap` — 空间分割最小重叠阈值
  - `spatial_storage` — 线程局部空间分割存储
  - `spatial_free_index` / `spatial_spin_lock` — 空间分割引用索引管理
  - `task_pool` — 多线程任务池
  - `unaligned_heuristic` — 非轴对齐启发式计算器
  - `build_mutex` — 构建互斥锁
- **关键方法**:
  - `run()` — 入口方法,执行完整构建流程并返回根节点
  - `add_references(BVHRange &root)` — 收集所有场景对象的图元引用
  - `add_reference_triangles(...)` — 添加三角形网格图元引用
  - `add_reference_curves(...)` — 添加毛发曲线图元引用
  - `add_reference_points(...)` — 添加点云图元引用
  - `add_reference_geometry(...)` — 通用几何体引用添加（分发到上述方法）
  - `add_reference_object(...)` — 添加对象实例引用
  - `build_node(const BVHRange &range, ...)` — 递归构建节点（支持空间分割）
  - `build_node(const BVHObjectBinning &range, int level)` — 递归构建节点（纯对象分箱）
  - `create_leaf_node(...)` — 创建叶节点
  - `create_object_leaf_nodes(...)` — 为对象创建叶节点
  - `thread_build_node(...)` — 多线程子任务:对象分箱构建
  - `thread_build_spatial_split_node(...)` — 多线程子任务:空间分割构建
  - `rotate(BVHNode *node, int max_depth, int iterations)` — 树旋转优化
  - `progress_update()` — 进度更新

## 核心函数

### `BVHBuild::run()`
1. 调用 `add_references()` 收集所有图元引用并计算根包围盒
2. 如果启用空间分割,计算 `spatial_min_overlap` 阈值(根据根包围盒面积和引用数量)
3. 根据是否启用空间分割选择 `build_node` 分支递归构建
4. 等待线程池中所有任务完成
5. 执行树旋转优化(`rotate`)
6. 更新可见性和时间范围
7. 将图元引用排列写入输出数组

### `BVHBuild::build_node`（空间分割版本）
1. 使用 `BVHMixedSplit` 同时计算对象分割和空间分割的 SAH 代价
2. 判断是否应创建叶节点(图元数量小于阈值或分割代价高于叶节点代价)
3. 选择代价更低的分割方式执行
4. 当图元数量超过 `THREAD_TASK_SIZE(4096)` 时,将子任务提交到线程池并行执行
5. 递归构建左右子节点
6. 对非轴对齐节点设置对齐变换空间

### `BVHBuild::build_node`（对象分箱版本）
1. 使用 `BVHObjectBinning` 计算分箱 SAH
2. 判断叶节点条件
3. 大任务提交到线程池并行
4. 递归构建左右子节点

### 运动模糊处理
`add_reference_triangles` / `add_reference_curves` / `add_reference_points` 中,当图元具有运动模糊时,会根据 `num_motion_*_steps` 将时间范围划分为多个子步,每个子步生成独立的图元引用,以提高运动图元的 BVH 质量。

## 依赖关系

- **内部头文件**:
  - `bvh/params.h` — `BVHParams`、`BVHRange`、`BVHReference` 等基础类型
  - `bvh/binning.h` — `BVHObjectBinning` 分箱器
  - `bvh/node.h` — `BVHNode`、`InnerNode`、`LeafNode` 节点类型
  - `bvh/split.h` — `BVHMixedSplit`、`BVHObjectSplit`、`BVHSpatialSplit` 分割策略
  - `bvh/unaligned.h` — `BVHUnaligned` 非轴对齐启发式
  - `scene/mesh.h`、`scene/hair.h`、`scene/pointcloud.h`、`scene/object.h` — 场景几何体
  - `util/task.h` — `TaskPool` 多线程任务池
  - `util/progress.h` — 进度报告
- **被引用**:
  - `bvh/bvh2.cpp` — `BVH2::build()` 创建 `BVHBuild` 实例执行构建
  - `bvh/node.cpp` — 引用构建相关参数
  - `bvh/split.h` / `bvh/split.cpp` — 分割策略友元访问 `BVHBuild` 成员
  - `scene/mesh.cpp` — 网格 BVH 构建

## 实现细节 / 关键算法

### 空间分割 (Spatial Split)
当对象间存在较大重叠时,纯对象分割可能产生较差的 BVH 质量。空间分割允许将一个图元同时"劈开"分配到两个子节点中(引用被复制),从而减少节点重叠。启用条件:
- `params.use_spatial_split` 为 true
- 候选分割的重叠面积大于 `spatial_min_overlap` 阈值
- 空间分割的 SAH 代价低于对象分割

### 多线程构建
当待分割范围的图元数大于 `THREAD_TASK_SIZE(4096)` 时,子任务会被提交到 `TaskPool` 异步执行。使用 `build_mutex` 保护共享数据。`enumerable_thread_specific<BVHSpatialStorage>` 提供线程局部的空间分割临时存储。

### 树旋转优化
构建完成后,对树执行多轮旋转(`rotate`),通过交换子节点来降低整棵树的 SAH 代价。旋转深度限制为 5 层,迭代次数最多 5 次。

### 非轴对齐节点
对于毛发曲线等几何体,`BVHUnaligned` 会计算最优对齐变换空间。当非轴对齐包围盒面积远小于轴对齐包围盒面积时(比值低于 `unaligned_split_threshold`),使用非轴对齐节点。

## 关联文件

- `bvh/binning.h` / `bvh/binning.cpp` — 对象分箱器
- `bvh/split.h` / `bvh/split.cpp` — 空间分割策略
- `bvh/node.h` / `bvh/node.cpp` — BVH 节点定义
- `bvh/bvh2.h` / `bvh/bvh2.cpp` — 二叉 BVH 打包器,调用本构建器
- `bvh/unaligned.h` / `bvh/unaligned.cpp` — 非轴对齐包围盒启发式
- `bvh/params.h` — BVH 参数和图元引用定义
