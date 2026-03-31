# light_tree_debug.h / light_tree_debug.cpp - 光源树调试可视化工具

## 概述

本文件提供了光源树的 Graphviz DOT 格式输出功能，用于调试和可视化光源树的层次结构。支持导出两种形式的树：构建阶段的 `LightTreeNode` 树和设备端的 `KernelLightTreeNode` 树。输出文件可使用 Graphviz 工具渲染为图形化的树状结构图。

## 类与结构体

本文件无类定义，仅提供自由函数。

## 核心函数

### light_tree_plot_to_file

```cpp
void light_tree_plot_to_file(const Scene &scene,
                             const LightTree &tree,
                             const LightTreeNode &root_node,
                             const string &filename);
```

- **功能**: 将构建阶段的光源树导出为 DOT 格式文件。输出包含节点信息（类型、光集成员资格、是否可共享）和发射体详情（类型、对象名称、包围盒、方向锥参数）。
- **输出格式**: 使用 record 形状节点，从左到右布局的有向图。节点间通过 `left`/`right`/`emitters` 端口连接。

### klight_tree_plot_to_file

```cpp
void klight_tree_plot_to_file(uint root_index,
                              const KernelLightTreeNode *knodes,
                              const string &filename);
```

- **功能**: 将设备端（内核）光源树导出为 DOT 格式文件。输出包含每个节点的包围盒、方向锥、能量、发射体信息、位路径和位跳过等内核数据。

### 静态辅助函数

- `get_node_id()` — 基于节点指针地址生成唯一节点标识符
- `get_emitter_id()` — 基于发射体指针地址生成唯一标识符
- `set_membership_str()` — 将光集成员位掩码转换为可读字符串（全部位为 1 时显示 "ALL"）
- `recursive_print_node()` — 递归输出节点定义（类型标签、光链接信息、子节点端口）
- `print_emitters()` — 输出所有发射体的详细信息（类型、对象名、光集成员、包围盒、方向锥）
- `recursive_print_node_relations()` — 递归输出节点之间的边连接关系
- `recursive_print_knode()` — 递归输出设备端节点及其连接关系

## 依赖关系

- **内部头文件**:
  - `scene/light_tree_debug.h`（自身头文件）
  - `scene/light.h` — 光源定义
  - `scene/light_tree.h` — 光源树数据结构
  - `scene/object.h` — 对象信息（用于获取名称）
  - `scene/scene.h` — 场景类
  - `util/path.h` — `path_fopen` 跨平台文件打开
  - `util/string.h` — `string_printf` 格式化
- **被引用**: 仅在 `scene/light_tree_debug.cpp` 中实现，通常由 `light.cpp` 中的调试代码调用

## 实现细节 / 关键算法

1. **DOT 格式输出**: 使用 Graphviz 的 `digraph` 格式，每个节点为 `record` 形状，字段包含节点类型标签、光链接信息以及子节点连接端口。有向边通过唯一的 `relation_id` 标识。

2. **节点信息展示**: 每个节点显示其类型组合（instance/leaf/inner/distant）、光集成员资格（具体值或 "ALL"）、是否可共享。叶子节点额外显示发射体连接端口。

3. **发射体信息展示**: 每个发射体显示其类型（light/triangle/mesh）、所属对象名称、光集成员、包围盒最小/最大角点、方向锥轴向和角度参数。

4. **设备端树展示**: 内核节点额外显示能量值、第一个发射体索引、发射体数量、位路径（bit_trail）和位跳过（bit_skip）等内核级别数据。

## 关联文件

- `scene/light_tree.h` / `scene/light_tree.cpp` — 光源树数据结构定义
- `scene/light.h` / `scene/light.cpp` — 光源管理器中调用调试输出
- `kernel/types.h` — `KernelLightTreeNode` 设备端节点结构定义
