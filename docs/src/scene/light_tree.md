# light_tree.h / light_tree.cpp - 光源树构建与多光源高效采样

## 概述

本文件实现了 Cycles 渲染器的光源树（Light Tree）数据结构，这是一种类似 BVH 的层次加速结构，用于高效地对大量光源进行重要性采样。光源树基于 PBRT 的 BVH 构建方法，使用表面积方向启发式（SAOH, Surface Area Orientation Heuristic）来评估分割代价，综合考虑包围盒、方向锥和能量信息。该结构支持本地光源、远距光源和发光网格三种类型的发射体。

## 类与结构体

### OrientationBounds

- **功能**: 描述光源法线方向的锥形边界，用于限定光源的朝向和发射范围。
- **关键成员**:
  - `axis` — 锥体的中心轴方向 (`float3`)
  - `theta_o` — 法线分布的边界角度（锥体半角）
  - `theta_e` — 发射轮廓的边界角度
- **关键方法**:
  - `calculate_measure()` — 基于论文公式计算方向度量值，用于 SAOH 启发式
  - `is_empty()` — 检查方向边界是否为空
- **特殊值**: `empty` 枚举设置最小值，确保与非空边界合并时返回非空结果

### LightTreeMeasure

- **功能**: 光源树节点的综合度量，包含包围盒、方向锥和能量，用于计算 SAOH 分割代价。
- **关键成员**:
  - `bbox` — 空间包围盒 (`BoundBox`)
  - `bcone` — 方向锥 (`OrientationBounds`)
  - `energy` — 聚合能量值 (`float`)
- **关键方法**:
  - `add(LightTreeMeasure&)` — 合并另一个度量值（扩展包围盒、合并方向锥、累加能量）
  - `calculate()` — 计算 SAOH 代价值 = `energy * area_measure * orientation_measure`
  - `transform(Transform&)` — 对度量值应用变换（仅支持均匀缩放）
  - `reset()` — 重置为默认空值

### LightTreeLightLink

- **功能**: 管理光链接（Light Linking）的位掩码系统，用于控制光源对特定接收器组的可见性。
- **关键成员**:
  - `set_membership` — 64 位成员位掩码
  - `shareable` — 当子树中所有发射体具有相同的光集成员资格时，该子树可被专用树共享
  - `shared_node_index` — 共享节点索引

### LightTreeEmitter

- **功能**: 光源树中的发射体，可以是内置光源、发光网格或发光三角形。
- **关键成员**:
  - `root` — 若为网格发射体，指向其子树根节点
  - `light_id` / `prim_id` — 联合体：光源设备索引或对象本地三角形索引
  - `object_id` — 对象索引
  - `centroid` — 发射体质心位置
  - `light_set_membership` — 光链接成员位掩码
  - `measure` — 发射体度量值
- **关键方法**:
  - 构造函数（网格版本）— 从 `Object` 创建网格发射体
  - 构造函数（光源/三角形版本）— 从场景数据计算完整的度量值，包括包围盒、方向锥和能量
  - `is_mesh()` / `is_triangle()` / `is_light()` — 类型判断

### LightTreeBucket

- **功能**: 构建过程中用于分割代价评估的桶结构，将发射体按质心位置分到 12 个桶中。
- **关键成员**:
  - `measure` — 桶内所有发射体的聚合度量
  - `light_link` — 桶内光链接信息
  - `count` — 桶内发射体数量
  - `num_buckets = 12` — 桶的固定数量

### LightTreeNode

- **功能**: 光源树节点，使用 `std::variant` 支持三种类型：叶子节点、内部节点和实例节点。
- **关键成员**:
  - `measure` — 节点聚合度量
  - `light_link` — 节点光链接信息
  - `bit_trail` — 位路径，用于树遍历
  - `type` — 节点类型位掩码（`LIGHT_TREE_INNER`、`LIGHT_TREE_LEAF`、`LIGHT_TREE_DISTANT`、`LIGHT_TREE_INSTANCE`）
- **内部结构**:
  - `Leaf` — 叶子节点，包含 `first_emitter_index` 和 `num_emitters`
  - `Inner` — 内部节点，包含两个子节点 `children[2]`
  - `Instance` — 实例节点，包含 `reference` 指向被引用的节点
- **关键方法**:
  - `make_leaf()` / `make_distant()` / `make_instance()` — 节点类型转换
  - `get_reference()` — 获取实例引用的原始节点
  - `is_instance()` / `is_leaf()` / `is_inner()` / `is_distant()` — 类型判断

### LightTree

- **继承**: 无（独立类）
- **功能**: 光源树的主类，负责构建和管理整个光源树结构。
- **关键成员**:
  - `root_` — 树根节点
  - `emitters_` — 所有发射体的扁平数组
  - `local_lights_` / `distant_lights_` / `mesh_lights_` — 按类型分组的临时发射体列表
  - `offset_map_` — 网格到三角形偏移的映射
  - `num_nodes` — 原子计数器，跟踪创建的节点数
  - `num_triangles` — 发光三角形总数
  - `light_link_receiver_used` — 使用的接收器光集的位掩码
  - `max_lights_in_leaf_` — 叶子节点最大光源数量
  - `task_pool` — 并行构建线程池
  - `MIN_EMITTERS_PER_THREAD = 4096` — 启动新线程的最小发射体数量阈值
- **关键方法**:
  - `build(Scene*, DeviceScene*)` — 构建完整的光源树，返回根节点指针
  - `create_node()` — 创建新节点并同步递增节点计数
  - `num_emitters()` / `get_emitters()` — 获取发射体数据
  - `recursive_build()` — 递归构建子树
  - `should_split()` — 评估是否应分割当前节点
  - `triangle_usable_as_light()` — 检查三角形是否可作为光发射体
  - `add_mesh()` — 将网格的所有发光三角形添加到发射体列表

## 核心函数

- `merge(OrientationBounds&, OrientationBounds&)` — 合并两个方向锥为包含两者的最小锥，使用球面线性插值（Slerp）计算新轴
- `operator+(LightTreeMeasure&, LightTreeMeasure&)` — 度量值加法
- `operator+(LightTreeBucket&, LightTreeBucket&)` — 桶加法
- `operator+(LightTreeLightLink&, LightTreeLightLink&)` — 光链接合并
- `sort_leaf()` — 按光链接掩码排序叶子节点中的发射体

## 依赖关系

- **内部头文件**:
  - `scene/light.h` — 光源定义
  - `scene/scene.h` — 场景类
  - `util/boundbox.h` — 包围盒
  - `util/task.h` — 并行任务池
  - `scene/mesh.h`、`scene/object.h`（cpp 中引用）
  - `util/math_fast.h`、`util/progress.h`（cpp 中引用）
- **被引用**: `scene/light.cpp`、`scene/light_tree_debug.cpp`、`scene/light_tree.cpp`

## 实现细节 / 关键算法

1. **SAOH（表面积方向启发式）**: 分割代价计算公式为 `cost = energy * area_measure * orientation_measure`，其中 `area_measure` 是包围盒面积（退化为长度当面积为零），`orientation_measure` 是方向锥的立体角度量。此启发式综合考虑空间局部性、方向一致性和能量分布。

2. **桶排序分割策略**: 在 3 个维度上各使用 12 个等分桶，将发射体按质心位置分配到桶中。通过累积前缀和后缀分别计算左右子树代价，选择代价最小的分割点。使用正则化因子 `max_extent * inv_extent` 避免偏向最长轴。

3. **并行递归构建**: 当子树发射体数量超过 `MIN_EMITTERS_PER_THREAD (4096)` 时，使用 `TaskPool` 将子树构建分发到新线程。使用 `nth_element` 进行高效的区间分割。

4. **三级树结构**: 顶层根节点的左子树包含所有本地光源和网格光源，右子树包含所有远距光源（作为叶节点）。网格光源各自拥有独立子树。

5. **网格实例化**: 相同网格的多个实例共享同一子树结构，通过 `Instance` 节点引用。每个实例可有不同的变换，通过 `LightTreeMeasure::transform()` 处理均匀缩放，非均匀缩放时重新计算。

6. **光链接优化**: 通过 `LightTreeLightLink::shareable` 标记子树是否可被不同光链接集的专用树共享，避免重复存储。

7. **发射体能量计算**: 三角形发射体的能量 = 面积 * 着色器发射估计值的平均值。各类光源根据类型做不同的辐照度到辐射度转换（如面光源除以 pi、点光源除以 4pi 等）。

## 关联文件

- `scene/light.h` / `scene/light.cpp` — 光源定义与管理器
- `scene/light_tree_debug.h` / `scene/light_tree_debug.cpp` — 光源树调试可视化
- `scene/mesh.h` — 网格数据访问
- `scene/object.h` — 对象变换与包围盒
- `kernel/types.h` — `LIGHT_TREE_INNER` 等节点类型常量
