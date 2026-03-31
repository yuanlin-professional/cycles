# node.h / node.cpp - 场景图节点基类与属性访问系统

## 概述

本文件定义了 Cycles 渲染器场景图（Scene Graph）中所有节点 (节点) 的基类 `Node`，以及节点所有权管理结构 `NodeOwner`。`Node` 提供了一套基于 `SocketType` 的通用属性（socket）读写接口，支持标量、向量、字符串、变换矩阵、枚举、节点引用等多种数据类型及其数组形式。该文件是整个场景图模块的基石——所有场景对象（相机、灯光、网格、着色器节点等）均继承自 `Node`。

设计上采用了"运行时类型信息 + 内存偏移量"的反射机制：每个 `Node` 持有一个指向 `NodeType` 的指针，通过 `SocketType::struct_offset` 直接操作结构体内存来实现泛型的属性读写，从而在不依赖 C++ 虚函数的情况下提供灵活的序列化、比较、哈希等能力。

## 类与结构体

### NodeOwner
- **功能**: 表示节点的所有者的抽象基类，仅提供虚析构函数。
- **关键成员**: 无数据成员。
- **关键方法**:
  - `virtual ~NodeOwner()` — 虚析构函数，确保多态删除安全。

### Node
- **功能**: 场景图中所有节点的抽象基类。管理节点的类型元信息、属性值读写、修改状态追踪、哈希计算、相等性比较以及引用计数。
- **关键成员**:
  - `name` (`ustring`) — 节点名称，默认使用类型名称，便于调试。
  - `type` (`const NodeType *`) — 指向节点类型元信息的指针，包含所有输入/输出 socket 定义。
  - `owner` (`const NodeOwner *`) — 指向所有者的指针（protected）。
  - `ref_count` (`int`) — 引用计数，用于管理节点生命周期（protected）。
  - `socket_modified` (`SocketModifiedFlags`) — 位掩码，追踪每个 socket 是否被修改（protected）。

- **关键方法**:
  - `set(const SocketType &input, T value)` — 一组重载函数，按类型设置 socket 值（bool、int、uint、uint64_t、float、float2、float3、ustring、Transform、Node* 以及对应的 array 版本）。设置时自动进行类型断言检查和脏标记。
  - `get_bool() / get_int() / get_float() / get_float3() / get_string() / get_transform() / get_node()` — 按类型获取 socket 值。
  - `get_bool_array() / get_int_array() / get_float_array()` 等 — 获取数组类型 socket 值。
  - `has_default_value(const SocketType &)` — 检查当前值是否为默认值（通过 memcmp 比较）。
  - `set_default_value(const SocketType &)` — 将 socket 重置为默认值（通过 memcpy）。
  - `equals_value(const Node &, const SocketType &)` — 比较两个节点的某个 socket 值是否相等。
  - `copy_value(const SocketType &, const Node &, const SocketType &)` — 从另一个节点拷贝 socket 值，处理 NODE 类型的引用计数。
  - `set_value(const SocketType &, const Node &, const SocketType &)` — 通过 `set()` 方法从另一个节点设置值（触发脏标记逻辑）。
  - `equals(const Node &)` — 比较两个同类型节点的所有输入 socket 值是否相等。
  - `hash(MD5Hash &)` — 基于节点类型名和所有 socket 值计算 MD5 哈希值。
  - `get_total_size_in_bytes()` — 计算节点所有 socket 占用的总内存大小（含数组实际数据）。
  - `is_a(const NodeType *)` — 类型检查，沿继承链向上查找是否匹配。
  - `socket_is_modified(const SocketType &)` — 检查指定 socket 是否被修改。
  - `is_modified()` — 检查是否有任何 socket 被修改。
  - `tag_modified()` — 将所有 socket 标记为已修改（设置所有位）。
  - `clear_modified()` — 清除所有修改标记。
  - `reference() / dereference() / reference_count() / clear_reference_count()` — 引用计数管理。
  - `set_owner() / get_owner()` — 所有者管理。
  - `dereference_all_used_nodes()` — 在析构时对所有 NODE / NODE_ARRAY 类型的引用进行递减（protected）。
  - `print_modified_sockets()` — 调试用，打印所有被修改的 socket 名称。

## 宏定义

### NODE_SOCKET_API_BASE_METHODS(type_, name, string_name)
为指定 socket 生成以下访问方法：
- `get_<name>_socket()` — 获取 socket 类型描述符（静态缓存）。
- `<name>_is_modified()` — 检查该 socket 是否被修改。
- `tag_<name>_modified()` — 手动标记该 socket 为已修改。
- `get_<name>()` — 获取 socket 的 const 引用值。

### NODE_SOCKET_API(type_, name)
在 `NODE_SOCKET_API_BASE` 基础上增加 `set_<name>(type_ value)` 设置方法。将成员变量声明为 `protected`，并生成完整的 getter/setter 接口。

### NODE_SOCKET_API_ARRAY(type_, name)
与 `NODE_SOCKET_API` 类似，但针对数组类型 socket，额外提供非 const 的 `get_<name>()` 引用返回。

### NODE_SOCKET_API_STRUCT_MEMBER(type_, name, member)
用于结构体成员风格的 socket（如 `name.member`），生成带点号名称的访问方法。

## 核心函数

### set_if_different() (protected, 模板)
- **签名**: `template<typename T> void set_if_different(const SocketType &input, T value)`
- **功能**: 仅当新值与当前值不同时才执行赋值，并设置 `modified_flag_bit`。这是所有 `set()` 方法的底层实现，避免不必要的脏标记。
- **Node* 特化**: 处理引用计数——旧节点 `dereference()`，新节点 `reference()`。
- **array<T> 特化**: 使用 `steal_data()` 进行零拷贝数据转移。
- **array<Node*> 特化**: 同时处理数组内所有节点的引用计数变化。

### hash()
- **签名**: `void hash(MD5Hash &md5)`
- **功能**: 将节点类型名称和所有输入 socket 的值追加到 MD5 哈希中。对 float3 类型特殊处理，仅哈希前 3 个分量（忽略第 4 个填充分量）。

### equals_value()
- **签名**: `bool equals_value(const Node &other, const SocketType &socket) const`
- **功能**: 根据 socket 类型分派到 `is_value_equal<T>` 或 `is_array_equal<T>` 模板函数进行逐类型比较。

## 依赖关系

- **内部头文件**:
  - `graph/node_type.h` — NodeType、SocketType 定义
  - `util/array.h` — 动态数组
  - `util/param.h` — ustring 参数类型
  - `util/types.h` — 基本类型 (float2, float3 等)
  - `util/md5.h` — MD5 哈希（.cpp）
  - `util/transform.h` — Transform 变换矩阵（.cpp）
- **外部库**: `<type_traits>`（标准库，用于枚举类型的 SFINAE 约束）
- **被引用**: 被 Cycles 中大量场景对象引用，包括：
  - `src/scene/` 下的 `background.h`、`camera.h`、`film.h`、`geometry.h`、`integrator.h`、`light.h`、`mesh.h`、`object.h`、`particles.h`、`pass.h`、`procedural.h`、`shader.h`、`shader_graph.h`、`shader_nodes.h`、`volume.h`
  - `src/session/buffers.h`、`src/session/tile.cpp`
  - `src/device/denoise.h`
  - `src/hydra/node_util.h`

## 实现细节 / 关键算法

1. **基于内存偏移的反射机制**: `get_socket_value<T>()` 通过将 `this` 指针转换为 `char*` 并加上 `socket.struct_offset` 来直接访问派生类的成员变量内存。这种方式避免了虚函数调用开销，使得属性的泛型访问非常高效。

2. **脏标记位掩码**: 每个 socket 在注册时被分配一个唯一的 `modified_flag_bit`（`1ull << index`），节点的 `socket_modified` 字段为 `uint64_t`，最多支持 64 个输入 socket 的独立脏追踪。

3. **引用计数管理**: 当 socket 类型为 `NODE` 或 `NODE_ARRAY` 时，`set_if_different()` 和 `copy_value()` 会自动管理被引用节点的引用计数，防止悬挂指针。`dereference_all_used_nodes()` 供派生类析构函数调用以正确清理。

4. **float3 哈希优化**: 在 MD5 哈希计算中，float3 类型仅哈希前 3 个 float 分量（`sizeof(float) * 3`），跳过第 4 个用于内存对齐的填充分量，确保哈希的确定性。

5. **数组值的零拷贝设置**: `set_if_different()` 的数组特化版本使用 `steal_data()` 将输入数组的数据直接移交给节点，避免深拷贝。

## 关联文件

- `src/graph/node_type.h` / `src/graph/node_type.cpp` — NodeType 和 SocketType 的完整定义与注册机制
- `src/graph/node_enum.h` — NodeEnum 枚举值映射
- `src/graph/node_xml.h` / `src/graph/node_xml.cpp` — 节点的 XML 序列化/反序列化
