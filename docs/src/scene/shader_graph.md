# shader_graph.h / shader_graph.cpp - 着色器节点图结构与图优化

## 概述

本文件定义了 Cycles 渲染器的着色器节点图（ShaderGraph）体系，包括着色器输入/输出插口（ShaderInput / ShaderOutput）、着色器节点基类（ShaderNode）以及着色器图（ShaderGraph）。着色器图是一个有向无环图(DAG)，节点之间通过输入/输出插口连接。ShaderGraph 负责图的构建、连接、优化（常量折叠、去重、死节点移除）以及从位移自动生成凹凸映射等图变换操作。

## 类与结构体

### ShaderInput

- **继承**: 无
- **功能**: 着色器节点的输入插口。可连接到某个输出插口，未连接时使用固定默认值或纹理坐标。
- **关键成员**:
  - `socket_type` — 插口类型描述（const SocketType&）
  - `parent` — 所属父节点（ShaderNode*）
  - `link` — 连接的输出插口（ShaderOutput*，nullptr 表示未连接）
  - `stack_offset` — 着色器虚拟机(SVM)编译器的栈偏移
  - `constant_folded_in` — 是否已被常量折叠（避免过度优化）
- **关键方法**:
  - `name()` — 返回 UI 显示名称
  - `type()` — 返回插口数据类型
  - `flags()` — 返回插口标志位
  - `set(float/float3/int)` — 设置默认值
  - `disconnect()` — 断开与输出插口的连接

### ShaderOutput

- **继承**: 无
- **功能**: 着色器节点的输出插口。可连接到多个输入插口。
- **关键成员**:
  - `socket_type` — 插口类型描述
  - `parent` — 所属父节点
  - `links` — 连接的所有输入插口列表（vector<ShaderInput*>）
  - `stack_offset` — SVM 栈偏移
- **关键方法**:
  - `name()` / `type()` — 名称和类型查询
  - `disconnect()` — 断开所有连接

### ShaderNode

- **继承**: `Node`（来自 `graph/node.h`）
- **功能**: 着色器图中节点的虚基类。所有具体节点类型（纹理节点、双向散射分布函数(BSDF)节点、数学节点等）均从此派生。
- **关键成员**:
  - `inputs` — 输入插口列表（unique_ptr_vector<ShaderInput>）
  - `outputs` — 输出插口列表（unique_ptr_vector<ShaderOutput>）
  - `id` — 图中的节点索引
  - `bump` — 凹凸映射采样位置（CENTER / DX / DY）
  - `bump_filter_width` — 凹凸映射滤波宽度
  - `special_type` — 特殊节点类型标识（PROXY / AUTOCONVERT / CLOSURE 等）
- **关键方法**:
  - `input(name)` / `output(name)` — 按名称查找输入/输出插口
  - `clone(ShaderGraph*)` — 纯虚方法，克隆节点
  - `compile(SVMCompiler&)` / `compile(OSLCompiler&)` — 纯虚方法，编译到 SVM 或 OSL
  - `expand(ShaderGraph*)` — 将节点展开为额外节点
  - `constant_fold(const ConstantFolder&)` — 常量折叠优化
  - `simplify_settings(Scene*)` — 简化艺术家设置为内核高效形式
  - `attributes(Shader*, AttributeRequestSet*)` — 声明所需的网格属性
  - `has_surface_emission()` / `has_surface_transparent()` / `has_surface_bssrdf()` — 特性查询
  - `has_spatial_varying()` — 是否具有空间变化特性
  - `has_volume_support()` — 是否支持体积渲染
  - `is_linear_operation()` — 是否为线性运算（用于体积采样优化）
  - `get_feature()` — 获取内核特性标志
  - `get_closure_type()` — 获取闭包类型
  - `equals(const ShaderNode&)` — 节点等价比较（用于去重）

### ShaderGraph

- **继承**: `NodeOwner`
- **功能**: 着色器节点图。管理节点集合、连接关系，并提供图简化、常量折叠、去重、死节点清理、凹凸映射生成、多重闭包变换等优化操作。
- **关键成员**:
  - `nodes` — 节点列表（unique_ptr_vector<ShaderNode>），第一个元素始终是 OutputNode
  - `num_node_ids` — 已分配的节点 ID 数量
  - `finalized` — 是否已完成最终化（只能最终化一次）
  - `simplified` — 是否已完成简化
  - `displacement_hash` — 位移子图的 MD5 哈希（用于检测位移变化）
- **关键方法**:
  - `output()` — 返回输出节点（OutputNode，总是 nodes[0]）
  - `connect(from, to)` — 连接两个插口，类型不匹配时自动插入转换节点（ConvertNode），闭包输入自动插入 EmissionNode
  - `disconnect(from/to)` — 断开连接
  - `relink(from, to)` — 重新链接插口（多个重载版本）
  - `create_node<T>(args...)` — 创建节点并添加到图中（模板方法）
  - `simplify(Scene*)` — 图简化流程：展开 -> 默认输入 -> 清理 -> 细化凹凸节点
  - `finalize(Scene*, do_bump, bump_in_object_space)` — 最终化：简化 + 凹凸映射生成 + 多重闭包变换
  - `remove_proxy_nodes()` — 移除代理节点（组导出时的临时节点）
  - `compute_displacement_hash()` — 计算位移子图的 MD5 哈希
  - `get_num_closures()` — 计算图中的闭包数量
  - `dump_graph(filename)` — 导出为 Graphviz DOT 格式（调试用）

## 枚举类型

- `ShaderBump` — 凹凸采样位置：NONE / CENTER / DX / DY
- `ShaderNodeSpecialType` — 特殊节点类型标识（12 种）：NONE / PROXY / AUTOCONVERT / GEOMETRY / OSL / IMAGE_SLOT / CLOSURE / COMBINE_CLOSURE / OUTPUT / BUMP / OUTPUT_AOV / LIGHT_PATH

## 辅助类型

- `ShaderNodeIDComparator` — 按节点 ID 排序的比较器
- `ShaderNodeSet` — 基于 ID 排序的节点集合（set<ShaderNode*, ShaderNodeIDComparator>）
- `ShaderNodeMap` — 节点到节点的映射（map<ShaderNode*, ShaderNode*, ShaderNodeIDComparator>）

## 宏定义

- `SHADER_NODE_CLASS(type)` — 标准着色器节点类声明宏（包含 NODE_DECLARE、构造函数、clone、compile）
- `SHADER_NODE_NO_CLONE_CLASS(type)` — 无默认 clone 实现的节点类声明宏
- `SHADER_NODE_BASE_CLASS(type)` — 基类节点声明宏（仅 clone 和 compile）

## 核心函数

### 图优化流水线（clean 方法调用顺序）

1. `constant_fold()` — 常量折叠：按拓扑序遍历，对每个节点的输出调用 `node->constant_fold()`
2. `simplify_settings()` — 简化设置：对每个节点调用 `node->simplify_settings()`
3. `deduplicate_nodes()` — 节点去重：自底向上遍历，合并设置和输入完全相同的节点
4. `optimize_volume_output()` — 体积输出优化：移除无意义的体积着色器，标记支持随机采样的属性节点
5. `break_cycles()` — 环路检测和打断（DFS）
6. 移除未被访问的无用节点

### 凹凸映射生成

- `bump_from_displacement()` — 从位移图自动生成凹凸映射：复制位移子图 3 份（center/dx/dy），分别标记不同采样位置，通过点积将位移向量转为高度值，连接到 BumpNode -> SetNormalNode
- `refine_bump_nodes()` — 细化 BumpNode：将 Height 输入的子图复制为 SampleCenter/SampleX/SampleY 三份

### 多重闭包变换
`transform_multi_closure()` — 将着色器的混合/添加闭包树转换为权重直接写入闭包节点的形式（避免构建闭包树再展平，改为直接写入数组）。对 MixClosureNode 插入 MixClosureWeightNode，对叶子闭包节点设置 SurfaceMixWeight/VolumeMixWeight。

## 依赖关系

- **内部头文件**:
  - `graph/node.h`, `graph/node_type.h` — 节点系统基类
  - `kernel/types.h` — 内核类型定义
  - `util/map.h`, `util/param.h`, `util/set.h`, `util/string.h`, `util/types.h`, `util/unique_ptr_vector.h`, `util/vector.h`
- **实现文件额外依赖**:
  - `scene/attribute.h`, `scene/constant_fold.h`, `scene/scene.h`, `scene/shader.h`, `scene/shader_nodes.h`
  - `util/algorithm.h`, `util/log.h`, `util/md5.h`, `util/queue.h`
- **被引用**: `scene/shader_nodes.h`, `scene/svm.h`, `scene/svm.cpp`, `scene/osl.h`, `scene/osl.cpp`, `scene/shader.cpp`, `scene/light.cpp`, `scene/mesh.cpp`, `scene/background.cpp`, `scene/constant_fold.cpp`, `test/render_graph_finalize_test.cpp`, `hydra/*.cpp`, `app/cycles_xml.cpp` 等 17 个文件

## 实现细节 / 关键算法

### 自动类型转换
`connect()` 方法在类型不匹配时自动插入转换节点：
- 目标为闭包(Closure)类型时：插入 EmissionNode（float 输入连到 Strength，其他连到 Color）
- 其他类型不匹配：插入 ConvertNode
- 闭包输出不能自动转换为其他类型

### 节点去重算法
`deduplicate_nodes()` 采用自底向上的拓扑遍历：
1. 从无依赖的叶子节点开始
2. 按节点类型名分组候选节点
3. 使用 `equals()` 方法比较节点（比较未链接插口的值和已链接插口的链接源）
4. 找到等价节点后，将所有输出重新链接到已有节点

### 体积输出优化
`optimize_volume_output()` 执行两项优化：
1. 检查体积输出是否包含有效的体积节点（`has_volume_support()`），如果没有则断开体积输出
2. 对沿非线性路径到达的 AttributeNode，禁用随机采样（`stochastic_sample = false`）

## 关联文件

- `src/scene/shader.h` / `shader.cpp` — 着色器和管理器
- `src/scene/shader_nodes.h` / `shader_nodes.cpp` — 所有具体节点类型
- `src/scene/constant_fold.h` / `constant_fold.cpp` — 常量折叠器
- `src/scene/svm.h` / `svm.cpp` — SVM 编译器（消费着色器图）
- `src/scene/osl.h` / `osl.cpp` — OSL 编译器（消费着色器图）
