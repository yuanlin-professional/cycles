# octree.h / octree.cpp - 体积渲染八叉树自适应步长结构

## 概述

本模块实现了用于体积渲染的八叉树（Octree）数据结构。八叉树根据体积密度差异对空间进行自适应细分，用于确定渲染体积时所需的步长大小。每个对象的每个着色器构建一棵八叉树，当节点内部的密度差异超过阈值时，节点会分裂为八个子节点。该结构支持并行构建和可视化调试。

## 类与结构体

### `OctreeNode`
八叉树的基础节点结构体。

| 成员 | 类型 | 说明 |
|------|------|------|
| `bbox` | `BoundBox` | 节点的包围盒 |
| `depth` | `int` | 节点在八叉树中的深度 |
| `sigma` | `Extrema<float>` | 节点内体积密度的最小值和最大值 |

- 提供 `visualize()` 方法，用于将节点可视化为字符串描述。

### `OctreeInternalNode`
继承自 `OctreeNode`，表示八叉树的内部节点（非叶节点）。

| 成员 | 类型 | 说明 |
|------|------|------|
| `children_` | `vector<shared_ptr<OctreeNode>>` | 8 个子节点的智能指针数组 |

### `Octree`
八叉树的核心管理类，负责构建、展平和可视化操作。

**公有成员:**

| 成员/方法 | 说明 |
|-----------|------|
| `Octree(const BoundBox &bbox)` | 构造函数，以包围盒初始化根节点 |
| `build(...)` | 根据体积密度构建八叉树（支持 OpenVDB） |
| `flatten(...)` | 将树状结构展平为数组，供内核上传使用 |
| `set_flattened()` / `is_flattened()` | 管理展平状态标志 |
| `flatten_index(x, y, z)` | 将 3D 网格坐标展平为 1D 索引 |
| `index_to_position(x, y, z)` | 从网格索引转换为体素左下角的世界坐标 |
| `voxel_size()` | 返回体素尺寸 |
| `get_num_nodes()` | 获取节点总数 |
| `get_root()` | 获取根节点 |
| `is_built()` | 查询是否已完成构建 |
| `visualize(...)` | 使用 Blender Python API 将八叉树绘制为线框盒 |

**私有成员:**

| 成员/方法 | 说明 |
|-----------|------|
| `resolution_` | 规则网格在各维度上的分辨率 |
| `sigmas_` | 网格中各体素的密度极值 |
| `get_extrema(...)` | 计算指定坐标范围内所有密度的极值（使用并行归约） |
| `evaluate_volume_density(...)` | 在网格中随机采样位置，评估着色器获取密度值 |
| `position_to_index_scale_` / `index_to_position_scale_` | 坐标与索引之间的缩放因子 |
| `should_split(...)` | 判断节点是否需要继续分裂 |
| `volume_scale(...)` | 缩放节点大小以使视口和最终渲染保持一致的细分级别 |
| `recursive_build(...)` | 递归构建节点及其子节点 |
| `make_internal(...)` | 将叶节点转换为内部节点，创建 8 个子包围盒 |
| `root_` | 根节点智能指针 |
| `num_nodes_` | 原子计数器，在构建过程中递增 |
| `task_pool_` | 用于并行构建的任务池 |

## 核心函数

### `Octree::build(...)`
八叉树的主构建入口。流程为：
1. 调用 `evaluate_volume_density()` 在设备上评估体积密度着色器
2. 计算 `volume_scale` 以处理对象空间与世界空间的缩放差异
3. 调用 `recursive_build()` 从根节点递归构建
4. 等待所有并行任务完成

### `Octree::should_split(...)`
分裂判定条件：当 `密度范围 * 节点对角线长度 * 缩放因子 > 1.442` 且深度未超过 `VOLUME_OCTREE_MAX_DEPTH` 时进行分裂。阈值 1.442 来自 Pixar 论文"Volume Rendering for Pixar's Elemental"。

### `Octree::flatten(...)`
以广度优先顺序将树结构展平为 `KernelOctreeNode` 数组。子节点连续存储，便于 GPU 内核高效遍历。

### `evaluate_volume_density(...)`
对体积密度场进行采样评估：
- 异质体积使用 `2^max_depth` 分辨率的 3D 网格
- 同质体积仅需要一个网格点
- 通过 `ShaderEval` 在设备端（GPU）进行密度着色器求值
- 支持 OpenVDB 内部遮罩优化，跳过网格外部的体素

### `fill_shader_input(...)` (静态函数)
为体积密度着色求值准备输入数据。每个体素需要两个 `KernelShaderEvalInput` 结构：第一个存储体素位置和对象 ID，第二个存储填充尺寸和着色器 ID。支持 OpenVDB 遮罩以跳过网格外部的区域。

### `read_shader_output(...)` (静态函数)
从设备读取密度求值的输出数据（最小值和最大值两个通道），并行写入 `sigmas` 数组。

### `Octree::make_internal(...)`
将一个普通节点转换为内部节点，生成 8 个子节点。每个子节点的包围盒通过对父节点包围盒的中点进行划分计算得出，使用位运算 `(i & 1, (i >> 1) & 1, (i >> 2) & 1)` 确定各子节点在八分空间中的位置。

### `Octree::visualize(...)`
生成 Blender Python 脚本，将八叉树的内部节点渲染为三个正交面的线框，用于调试目的。

## 依赖关系

- **内部头文件**:
  - `util/boundbox.h` - 包围盒工具
  - `util/task.h` - 并行任务池
  - `util/log.h` - 日志
  - `util/progress.h` - 进度报告
  - `scene/object.h` - 场景对象
  - `scene/volume.h` - 体积管理器
  - `integrator/shader_eval.h` - 着色器评估
- **外部依赖**: OpenVDB（条件编译 `WITH_OPENVDB`）
- **被引用**: `scene/volume.cpp`

## 实现细节 / 关键算法

### 自适应分裂策略
分裂阈值基于 Pixar 的 Elemental 渲染论文，公式为 `sigma_range * bbox_diagonal * scale > 1.442`。这意味着：
- 密度变化剧烈的区域会被细分为更小的节点，从而使用更小的步长
- 密度均匀的区域保持大节点，减少采样开销
- 深度上限为 `VOLUME_OCTREE_MAX_DEPTH`

### 并行构建
使用 `TaskPool` 实现并行递归构建。每个子节点的构建被推入任务池，多线程同时处理不同分支。节点计数使用 `std::atomic<int>` 保证线程安全。

### 密度评估流程
通过 `ShaderEval` 在 GPU 上批量评估体积着色器，获取每个体素的密度最小值和最大值。每个体素略微膨胀（0.2 倍体素尺寸）以确保不遗漏边界特征。

### 展平策略
采用广度优先遍历将树展平为一维数组，8 个兄弟节点在数组中连续存储（由 `first_child` 索引指向），确保内核遍历时的缓存友好性。

### OpenVDB 集成
通过 `vdb_voxel_intersect()` 使用 OpenVDB 的 `FindActiveValues` 判断体素是否位于网格内部，网格外部的体素密度直接设为零，避免不必要的着色器求值。

## 关联文件

- `scene/volume.h` / `scene/volume.cpp` - 体积管理器，调用八叉树构建
- `integrator/shader_eval.h` - 设备端着色器评估接口
- `kernel/types.h` - 定义 `KernelOctreeNode` 内核数据结构
- `util/boundbox.h` - `BoundBox` 和 `Extrema<float>` 类型定义
- `util/task.h` - `TaskPool` 并行任务管理
