# node_type.h / node_type.cpp - 节点类型系统与 Socket 定义

## 概述

本文件定义了 Cycles 渲染器节点 (节点) 类型系统的核心基础设施，包括 `SocketType`（socket 类型描述符）和 `NodeType`（节点类型注册表）。它提供了一套运行时类型反射机制，使得节点的输入/输出端口可以在运行时被发现、注册和查询。该文件还包含大量宏定义，用于在派生类中以声明式的方式注册 socket（属性端口）和定义节点类型，构成了整个场景图模块的类型元信息层。

## 类与结构体

### SocketType
- **功能**: 描述一个节点 socket（输入或输出端口）的完整元信息，包括名称、数据类型、在结构体中的内存偏移、默认值、标志位等。
- **关键成员**:
  - `name` (`ustring`) — socket 的内部名称（用于程序化访问）。
  - `ui_name` (`ustring`) — socket 的 UI 显示名称。
  - `type` (`Type`) — socket 的数据类型枚举值。
  - `struct_offset` (`int`) — socket 值在所属 Node 结构体中的内存偏移量。
  - `default_value` (`const void *`) — 指向默认值的指针。
  - `enum_values` (`const NodeEnum *`) — 仅 ENUM 类型有效，指向枚举值映射表。
  - `node_type` (`const NodeType *`) — 仅 NODE/NODE_ARRAY 类型有效，指向允许的节点类型。
  - `flags` (`int`) — 标志位组合（可链接、可动画、内部专用等）。
  - `modified_flag_bit` (`SocketModifiedFlags`) — 在脏标记位掩码中对应的位。

- **关键方法**:
  - `storage_size()` — 返回该类型在内存中的存储大小（非压缩）。
  - `packed_size()` — 返回该类型的紧凑（packed）大小，用于序列化比较。对于 float3 类型，packed 大小为 `sizeof(packed_float3)`，而 storage 大小为 `sizeof(float3)`。
  - `is_array()` — 判断类型是否为数组类型（`type >= BOOLEAN_ARRAY`）。
  - `size(Type, bool packed)` (静态) — 根据类型和是否紧凑返回大小。
  - `max_size()` (静态) — 返回所有类型中的最大存储大小（`sizeof(Transform)`）。
  - `type_name(Type)` (静态) — 返回类型的字符串名称。
  - `zero_default_value()` (静态) — 返回指向全零 Transform 的指针，作为通用零值默认值。
  - `is_float3(Type)` (静态) — 判断类型是否为 float3 家族（COLOR、VECTOR、POINT、NORMAL）。

### NodeType
- **功能**: 描述一种节点类型的完整元信息，包含该类型的所有输入/输出 socket 定义、创建函数、类型名称和继承关系。同时管理全局节点类型注册表。
- **关键成员**:
  - `name` (`ustring`) — 节点类型名称。
  - `type` (`Type`) — 节点类别（NONE 或 SHADER）。
  - `base` (`const NodeType *`) — 基类类型指针，用于类型继承。
  - `inputs` (`vector<SocketType>`) — 所有输入 socket 的描述符列表。
  - `outputs` (`vector<SocketType>`) — 所有输出 socket 的描述符列表。
  - `create` (`CreateFunc`) — 工厂函数指针，用于创建该类型节点的实例。

- **关键方法**:
  - `register_input(...)` — 注册一个输入 socket。自动分配 `modified_flag_bit`（`1ull << inputs.size()`），断言不超过 64 个输入。
  - `register_output(...)` — 注册一个输出 socket，默认带 `LINKABLE` 标志。
  - `find_input(ustring name)` — 按名称查找输入 socket，线性搜索。
  - `find_output(ustring name)` — 按名称查找输出 socket，线性搜索。
  - `add(const char *name, CreateFunc, Type, const NodeType *base)` (静态) — 向全局注册表中添加一个新的节点类型。如果重复注册会输出错误日志。
  - `find(ustring name)` (静态) — 从全局注册表中按名称查找节点类型。
  - `types()` (静态) — 返回全局节点类型注册表（`unordered_map<ustring, NodeType>`），使用函数局部静态变量确保初始化顺序。

## 枚举与常量

### SocketType::Type
socket 数据类型枚举，包含：
- 标量类型：`BOOLEAN`、`FLOAT`、`INT`、`UINT`、`UINT64`
- 向量类型：`COLOR`、`VECTOR`、`POINT`、`NORMAL`、`POINT2`
- 特殊类型：`CLOSURE`、`STRING`、`ENUM`、`TRANSFORM`、`NODE`
- 数组类型：上述各类型的 `_ARRAY` 版本
- 辅助值：`UNDEFINED`（0）、`NUM_TYPES`（总数哨兵）

### SocketType::Flags
socket 标志位枚举：
- `LINKABLE` (1<<0) — 可被连接（作为着色器图的可连接端口）
- `ANIMATABLE` (1<<1) — 可被动画驱动
- `SVM_INTERNAL` (1<<2) — 着色器虚拟机 (SVM) 内部使用
- `OSL_INTERNAL` (1<<3) — 开放着色语言 (OSL) 内部使用
- `INTERNAL` — SVM_INTERNAL | OSL_INTERNAL 的组合
- `LINK_TEXTURE_GENERATED` 至 `LINK_OSL_INITIALIZER` (1<<4 至 1<<12) — 控制默认链接行为
- `DEFAULT_LINK_MASK` — 所有默认链接标志的组合掩码

### NodeType::Type
节点类别枚举：
- `NONE` — 通用节点
- `SHADER` — 着色器节点

## 宏定义

### 节点定义宏

- **`NODE_DECLARE`** — 在类声明中使用，声明 `get_node_type()`、`register_type<T>()`、`create()` 等静态成员。
- **`NODE_DEFINE(structname)`** — 在类实现中使用，定义节点类型的工厂方法和延迟注册逻辑。采用双重检查锁定（Double-Checked Locking）模式确保线程安全的单次初始化。
- **`NODE_ABSTRACT_DECLARE`** — 用于抽象基类的声明。
- **`NODE_ABSTRACT_DEFINE(structname)`** — 用于抽象基类的实现，基类类型在 getter 函数中通过局部静态变量构造，确保正确的初始化顺序。

### Socket 定义宏

- **`SOCKET_DEFINE(name, ui_name, default_value, datatype, TYPE, flags, ...)`** — 通用 socket 注册宏，使用 `static_assert` 验证类型匹配，通过 `SOCKET_OFFSETOF` 计算内存偏移。
- **`SOCKET_BOOLEAN / SOCKET_INT / SOCKET_UINT / SOCKET_FLOAT / SOCKET_COLOR / SOCKET_VECTOR / SOCKET_POINT / SOCKET_NORMAL / SOCKET_POINT2 / SOCKET_STRING / SOCKET_TRANSFORM`** — 各基本类型的快捷注册宏（不可链接）。
- **`SOCKET_ENUM(name, ui_name, values, default_value, ...)`** — 枚举类型注册宏，需额外提供 `NodeEnum` 引用。
- **`SOCKET_NODE(name, ui_name, node_type, ...)`** — 节点引用类型注册宏，需指定允许的 `NodeType`。
- **`SOCKET_*_ARRAY`** — 各类型的数组版本注册宏。
- **`SOCKET_IN_BOOLEAN / SOCKET_IN_INT / SOCKET_IN_FLOAT / ...`** — 带 `LINKABLE` 标志的输入 socket 注册宏，用于着色器节点图中可连接的端口。
- **`SOCKET_IN_CLOSURE`** — 闭包输入 socket 注册宏（特殊处理，无默认值和偏移）。
- **`SOCKET_OUT_BOOLEAN / SOCKET_OUT_FLOAT / SOCKET_OUT_COLOR / ...`** — 输出 socket 注册宏。

## 核心函数

### NodeType::register_input()
- **签名**: `void register_input(ustring name, ustring ui_name, SocketType::Type type, int struct_offset, const void *default_value, const NodeEnum *enum_values, const NodeType *node_type, int flags, int extra_flags)`
- **功能**: 构造一个 `SocketType` 并追加到 `inputs` 列表。自动分配 `modified_flag_bit = (1ull << inputs.size())`，确保每个 socket 拥有唯一的脏标记位。`flags` 和 `extra_flags` 按位或合并。

### NodeType::add()
- **签名**: `static NodeType *add(const char *name, CreateFunc create, Type type, const NodeType *base)`
- **功能**: 在全局注册表中创建并注册新的节点类型。如果已存在同名类型，输出错误日志并返回 nullptr。如果提供了 `base`，则从基类继承所有已注册的 socket。

### SocketType::size()
- **签名**: `static size_t size(Type type, bool packed)`
- **功能**: 返回给定 socket 类型的字节大小。当 `packed=true` 时，float3 家族类型返回 `sizeof(packed_float3)`；当 `packed=false` 时返回 `sizeof(float3)`。数组类型返回 `sizeof(array<T>)`（仅容器头大小，不含数据）。

## 依赖关系

- **内部头文件**:
  - `graph/node_enum.h` — NodeEnum 枚举映射
  - `util/array.h` — 动态数组类型
  - `util/map.h` — unordered_map
  - `util/param.h` — ustring
  - `util/thread.h` — 线程互斥锁（用于 NODE_DEFINE 中的双重检查锁定）
  - `util/unique_ptr.h` — unique_ptr 和 make_unique
  - `util/vector.h` — vector
  - `util/log.h`（.cpp）— 日志输出
  - `util/transform.h`（.cpp）— Transform 和 transform_zero()
  - `util/types_float3.h`（.cpp）— packed_float3
- **外部库**: 无
- **被引用**:
  - `src/graph/node.h` — Node 基类依赖 NodeType
  - `src/graph/node.cpp` — Node 实现
  - `src/graph/node_type.cpp` — 本文件自身实现
  - `src/scene/shader_graph.h` — 着色器图

## 实现细节 / 关键算法

1. **全局注册表模式**: `NodeType::types()` 使用函数局部静态变量返回全局 `unordered_map`，解决了 C++ 静态初始化顺序问题（Static Initialization Order Fiasco）。所有节点类型通过 `NODE_DEFINE` 宏在首次访问时自动注册。

2. **双重检查锁定**: `NODE_DEFINE` 宏生成的 `get_node_type()` 方法先检查 `node_type_` 是否为 nullptr，不为空则直接返回；否则获取互斥锁后再次检查，确保多线程环境下只初始化一次。

3. **继承机制**: `NodeType` 构造函数接受可选的 `base` 指针。当提供基类时，自动拷贝基类的 `inputs` 和 `outputs` 列表，使派生类型继承基类的所有 socket 定义。

4. **modified_flag_bit 分配**: 每个输入 socket 按注册顺序分配 `1ull << index` 作为其脏标记位。由于 `SocketModifiedFlags` 为 `uint64_t`，最多支持 64 个独立追踪的输入 socket，注册时通过 `assert` 检查不超过此限制。

5. **packed vs storage 大小**: float3 类型在内存中通常为 16 字节对齐（`sizeof(float3)` = 16），但其有效数据仅 12 字节（`sizeof(packed_float3)` = 12）。`packed_size()` 用于序列化和默认值比较，`storage_size()` 用于值拷贝，两者的区分确保了正确性。

## 关联文件

- `src/graph/node.h` / `src/graph/node.cpp` — 使用 NodeType 和 SocketType 的节点基类
- `src/graph/node_enum.h` — NodeEnum 枚举映射工具
- `src/graph/node_xml.h` / `src/graph/node_xml.cpp` — 基于 SocketType 的 XML 序列化
- `src/scene/shader_graph.h` — 着色器图，使用 NodeType 管理着色器节点
