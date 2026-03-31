# graph - 节点图系统

## 概述

`graph` 模块是 Cycles 渲染引擎的基础数据模型层，提供了一个通用的、基于类型反射的节点系统（Node System）。它定义了所有场景对象（如相机、光源、几何体、着色器、积分器、胶片等）的公共基类 `Node`，以及围绕该基类构建的类型注册、Socket 定义、属性存取和序列化机制。

在 Cycles 架构中，`graph` 模块处于 **场景层（`scene`）与工具层（`util`）之间** 的核心位置：

- 向上为 `scene` 模块中的所有场景元素（`Camera`、`Light`、`Shader`、`ShaderNode`、`Geometry`、`Object`、`Film`、`Integrator` 等）提供统一的属性声明、读写、比较、哈希和修改追踪能力。
- 向下仅依赖 `util` 模块的基础数据类型和容器。

该模块并非着色器节点图的"连接拓扑"管理器（连接关系由 `scene/shader_graph.h` 中的 `ShaderGraph` 负责），而是更底层的 **节点属性系统**——即每个节点拥有哪些 Socket（输入/输出端口）、这些 Socket 的类型与默认值如何定义、属性值如何存取和序列化。

## 目录结构

| 文件名 | 类型 | 说明 |
|--------|------|------|
| `CMakeLists.txt` | 构建 | 构建配置，编译为 `cycles_graph` 库，依赖 `cycles_util` |
| `node.h` | 头文件 | `Node` 基类与 `NodeOwner` 的定义；`NODE_SOCKET_API` 系列宏 |
| `node.cpp` | 源文件 | `Node` 的全部实现：属性存取、比较、哈希、修改追踪、引用计数 |
| `node_type.h` | 头文件 | `SocketType`、`NodeType` 的定义；`NODE_DECLARE`/`NODE_DEFINE` 宏；全套 `SOCKET_*` 定义宏 |
| `node_type.cpp` | 源文件 | `SocketType` 尺寸计算、`NodeType` 注册/查找实现、全局类型注册表 |
| `node_enum.h` | 头文件 | `NodeEnum` 枚举双向映射工具类 |
| `node_xml.h` | 头文件 | XML 序列化/反序列化接口声明（需要 `WITH_PUGIXML`） |
| `node_xml.cpp` | 源文件 | XML 读写实现，支持所有 Socket 类型的文本化读写 |

## 核心类与数据结构

### SocketType

- **定义位置**: `node_type.h`
- **功能**: 描述节点上单个输入或输出端口（Socket）的完整元数据。
- **关键成员**:
  - `name` / `ui_name` — Socket 内部名称与 UI 显示名称
  - `type` — 数据类型枚举（`BOOLEAN`、`FLOAT`、`INT`、`UINT`、`UINT64`、`COLOR`、`VECTOR`、`POINT`、`NORMAL`、`POINT2`、`CLOSURE`、`STRING`、`ENUM`、`TRANSFORM`、`NODE`，以及对应的 `*_ARRAY` 变体，共 26 种）
  - `struct_offset` — 该 Socket 值在 Node 结构体中的字节偏移量（用于通用存取）
  - `default_value` — 指向默认值的指针
  - `enum_values` — 指向 `NodeEnum` 的指针（仅 `ENUM` 类型使用）
  - `node_type` — 指向引用的 `NodeType`（仅 `NODE`/`NODE_ARRAY` 类型使用）
  - `flags` — 标志位组合，包括 `LINKABLE`（可连接）、`ANIMATABLE`（可动画）、`SVM_INTERNAL` / `OSL_INTERNAL`（着色器虚拟机 (SVM) / 开放着色语言 (OSL) 内部专用）、`LINK_TEXTURE_*` 等默认连接提示
  - `modified_flag_bit` — 用于修改追踪的位掩码（每个 Socket 占用 `uint64_t` 中的一位，因此单个节点最多支持 64 个输入 Socket）
- **关键方法**:
  - `storage_size()` / `packed_size()` — 返回该类型在内存/紧凑存储中的字节大小
  - `is_array()` — 判断是否为数组类型
  - `is_float3()` — 判断是否为 float3 系列类型（COLOR / VECTOR / POINT / NORMAL）

### NodeType

- **定义位置**: `node_type.h`
- **功能**: 描述一种节点类型的完整元信息，包括该类型拥有的所有输入/输出 Socket、工厂函数和继承关系。类似于运行时的"类描述符"或"类型反射信息"。
- **关键成员**:
  - `name` — 类型名称（如 `"camera"`、`"diffuse_bsdf"` 等）
  - `type` — `NONE` 或 `SHADER`（区分普通节点与着色器节点）
  - `base` — 指向父 `NodeType`（支持继承，派生类自动继承基类的 Socket 定义）
  - `inputs` / `outputs` — `SocketType` 的向量，存储所有注册的输入和输出端口
  - `create` — 工厂函数指针 (`CreateFunc`)，用于动态创建该类型的节点实例
- **关键方法**:
  - `register_input()` — 注册一个输入 Socket（在 `NODE_DEFINE` 宏展开的 `register_type` 函数中调用）
  - `register_output()` — 注册一个输出 Socket
  - `find_input()` / `find_output()` — 按名称查找 Socket
  - `add()` — 静态方法，向全局注册表添加一个新的节点类型
  - `find()` — 静态方法，从全局注册表按名称查找节点类型
  - `types()` — 静态方法，返回全局类型注册表 (`unordered_map<ustring, NodeType>`)

### Node

- **定义位置**: `node.h`
- **功能**: 所有场景对象的抽象基类。每个 Node 实例都关联一个 `NodeType`，后者描述该实例拥有的全部 Socket。Node 提供了基于 `SocketType` 的通用属性存取接口。
- **关键成员**:
  - `name` — 节点实例名称
  - `type` — 指向该节点的 `NodeType` 描述
  - `owner` — 指向拥有该节点的 `NodeOwner`
  - `socket_modified` — `uint64_t` 位域，追踪哪些 Socket 被修改过
  - `ref_count` — 引用计数（用于 `NODE` / `NODE_ARRAY` 类型的 Socket 引用管理）
- **关键方法**:
  - `set()` — 一组重载函数，按类型设置 Socket 值（内部调用 `set_if_different`，仅在值变化时标记修改位）
  - `get_bool()` / `get_int()` / `get_float()` / `get_float3()` / `get_string()` 等 — 按类型获取 Socket 值
  - `equals()` / `equals_value()` — 比较两个同类型节点的所有 Socket 值是否相等
  - `hash()` — 计算节点的 MD5 哈希（包含类型名和所有 Socket 值）
  - `copy_value()` / `set_value()` — 在不同节点间复制或设置 Socket 值
  - `is_a()` — 运行时类型检查（沿 `NodeType::base` 链向上遍历）
  - `socket_is_modified()` / `is_modified()` / `tag_modified()` / `clear_modified()` — 修改追踪接口
  - `reference()` / `dereference()` — 引用计数管理
  - `dereference_all_used_nodes()` — 在析构时对所有引用的子节点解引用

### NodeOwner

- **定义位置**: `node.h`
- **功能**: 节点所有者的基类接口。被 `Scene` 和 `ShaderGraph` 等类继承，用于标识谁"拥有"某个 Node 实例。
- **关键成员**: 仅包含一个虚析构函数。

### NodeEnum

- **定义位置**: `node_enum.h`
- **功能**: 枚举值的双向映射工具。内部维护 `ustring -> int` 和 `int -> ustring` 两个哈希映射，用于枚举类型 Socket 的名称-值互转。
- **关键方法**:
  - `insert()` — 注册枚举项（名称与整数值的映射）
  - `exists()` — 检查名称或整数值是否存在
  - `operator[]` — 按名称获取整数值，或按整数值获取名称

### XMLReader

- **定义位置**: `node_xml.h`（需要 `WITH_PUGIXML` 编译条件）
- **功能**: XML 反序列化时的上下文，维护节点名称到 `Node*` 的映射表，用于解析 `NODE`/`NODE_ARRAY` 类型 Socket 中的引用关系。

## 模块架构

### 类型注册与反射机制

模块的核心设计模式是 **编译期声明 + 运行期注册** 的类型反射系统：

1. **声明阶段** — 派生类在头文件中使用 `NODE_DECLARE` 宏声明静态类型信息。对于抽象基类使用 `NODE_ABSTRACT_DECLARE`。

2. **注册阶段** — 派生类在源文件中使用 `NODE_DEFINE(ClassName)` 宏定义 `register_type<T>()` 函数体。该函数体内使用 `SOCKET_*` 系列宏（如 `SOCKET_FLOAT`、`SOCKET_COLOR`、`SOCKET_ENUM` 等）逐一注册输入 Socket，使用 `SOCKET_OUT_*` 宏注册输出 Socket。`NODE_DEFINE` 宏利用静态变量的初始化来触发自动注册，并通过双重检查锁（double-checked locking）保证线程安全。

3. **运行期** — `NodeType::types()` 维护一个全局 `unordered_map<ustring, NodeType>`，所有注册的节点类型均可通过 `NodeType::find()` 按名称查找。

### 属性存取机制

`Node` 的属性存取基于 **struct offset（结构体偏移量）** 实现：

- 每个 `SocketType` 记录了该属性在 Node 子类结构体中的字节偏移（通过 `SOCKET_OFFSETOF` 宏在注册时计算）。
- `get_socket_value<T>(node, socket)` 通过 `(T*)((char*)node + socket.struct_offset)` 直接访问内存。
- 这种设计避免了虚函数或回调的开销，同时支持完全通用的序列化、比较和哈希。

### 修改追踪系统

每个 Socket 在注册时被分配 `uint64_t` 中的一个位（`modified_flag_bit = 1ull << index`）。`set_if_different()` 仅在新值与旧值不同时设置修改标记位。`scene` 层据此判断哪些属性被用户修改过，从而实现增量更新。

### 引用计数

对于 `NODE` 和 `NODE_ARRAY` 类型的 Socket（即一个节点引用另一个节点），`set_if_different()` 包含专门的重载版本，在赋值时自动对旧节点 `dereference()`、对新节点 `reference()`。析构时通过 `dereference_all_used_nodes()` 统一解引用。

### 继承关系

```
NodeOwner (虚基类，节点所有者)
├── Scene         (场景，拥有场景级节点)
├── ShaderGraph   (着色器图，拥有着色器节点)
└── Procedural    (程序化对象，同时也是 Node)

Node (抽象基类，所有场景对象)
├── Camera        (相机)
├── Light         (光源)
├── Film          (胶片/输出设置)
├── Integrator    (积分器)
├── Background    (环境背景)
├── Shader        (着色器)
├── Object        (场景对象)
├── Geometry      (几何体基类)
│   ├── Mesh        (网格)
│   ├── Hair        (毛发)
│   ├── PointCloud  (点云)
│   └── Volume      (体积)
├── Pass          (渲染通道)
├── ParticleSystem (粒子系统)
├── ShaderNode    (着色器节点基类，ShaderGraph 中的各种节点)
├── DenoiseParams (降噪参数)
├── BufferPass    (缓冲通道)
├── BufferParams  (缓冲参数)
└── Procedural    (程序化对象，同时也是 NodeOwner)
```

### Socket 类型体系

```
基础标量类型          数组类型
─────────────        ──────────────
BOOLEAN              BOOLEAN_ARRAY
FLOAT                FLOAT_ARRAY
INT                  INT_ARRAY
UINT                 (无数组变体)
UINT64               (无数组变体)
COLOR    (float3)    COLOR_ARRAY
VECTOR   (float3)    VECTOR_ARRAY
POINT    (float3)    POINT_ARRAY
NORMAL   (float3)    NORMAL_ARRAY
POINT2   (float2)    POINT2_ARRAY
STRING   (ustring)   STRING_ARRAY
ENUM     (int)       (无数组变体)
TRANSFORM            TRANSFORM_ARRAY
NODE     (Node*)     NODE_ARRAY
CLOSURE              (无数组变体，大小为 0)
```

## 依赖关系

### 上游依赖（本模块依赖）

| 模块 | 用途 |
|------|------|
| `util` (`cycles_util`) | 基础类型（`ustring`、`float2`、`float3`、`Transform`、`array`）、容器（`vector`、`map`、`unordered_map`）、线程互斥锁（`thread_mutex`）、MD5 哈希、XML 工具（pugixml 封装） |

### 下游依赖（依赖本模块）

| 模块 | 使用方式 |
|------|----------|
| `scene` | 所有场景对象类（`Camera`、`Light`、`Shader`、`ShaderNode`、`Geometry`、`Mesh`、`Object`、`Film`、`Integrator`、`Background`、`Pass`、`ParticleSystem`、`Procedural` 等）均继承自 `Node`；`ShaderGraph` 和 `Scene` 继承自 `NodeOwner` |
| `device` | `DenoiseParams` 继承自 `Node` |
| `session` | `BufferPass`、`BufferParams` 继承自 `Node`；`tile.cpp` 使用节点类型 |
| `hydra` | `node_util.h` 使用节点类型系统进行 Hydra 代理的属性映射 |
| `app` | `cycles_xml.cpp` 使用 XML 序列化接口读写场景文件 |
| `test` | 测试代码链接 `cycles_graph` 库 |

## 关键算法与实现细节

### 基于 struct offset 的通用属性存取

属性存取的核心是通过 `struct_offset` 进行指针运算直接读写节点成员：

```cpp
template<typename T>
static T &get_socket_value(const Node *node, const SocketType &socket) {
    return (T &)*(((char *)node) + socket.struct_offset);
}
```

这要求 `SOCKET_OFFSETOF(T, name)` 宏（实际为 `offsetof`）在编译期计算派生类成员的偏移量，并在注册时存入 `SocketType::struct_offset`。运行时即可无需虚函数调用直接访问任意 Socket 值。

### set_if_different 的惰性修改标记

`set_if_different` 是所有 `set()` 方法的内部实现核心。它先比较新值与当前值是否相等，仅在不等时才赋值并设置 `socket_modified` 的对应位。这一设计使得 `scene` 层可以通过 `socket_is_modified()` 高效判断哪些属性发生了变化，避免不必要的重新编译着色器或重建加速结构。

对于数组类型，比较开销可能较大，因此当 Socket 已被标记为已修改时跳过比较，直接使用 `steal_data()` 接管数组内存所有权。

### MD5 哈希计算

`Node::hash()` 遍历所有输入 Socket，将类型名和每个 Socket 的名称及值追加到 MD5 上下文中。对于 float3 类型，仅哈希前 3 个分量（跳过第 4 个填充分量），保证在不同对齐下哈希结果一致。

### 节点类型的线程安全自动注册

`NODE_DEFINE` 宏生成的 `get_node_type()` 使用双重检查锁模式：

```cpp
const NodeType *StructName::get_node_type() {
    if (node_type_ == nullptr) {
        thread_scoped_lock lock(node_type_mutex_);
        if (node_type_ == nullptr) {
            node_type_ = StructName::register_type<StructName>();
        }
    }
    return node_type_;
}
```

静态变量 `node_type_` 的初始化会在程序启动时触发注册，同时通过互斥锁避免多线程竞争。

### XML 序列化

`node_xml.cpp` 实现了节点属性与 XML 属性之间的双向转换：

- **读取**: `xml_read_node()` 遍历节点类型的所有输入 Socket，从 XML 属性中解析对应值。对于 `NODE`/`NODE_ARRAY` 类型，通过 `XMLReader::node_map` 解析名称引用。
- **写入**: `xml_write_node()` 遍历所有输入 Socket，跳过内部 Socket 和保持默认值的 Socket，将其余属性写为 XML 属性。

## 参见

- `src/scene/shader_graph.h` — `ShaderGraph` 和 `ShaderNode`，基于 `Node`/`NodeOwner` 构建的着色器节点图连接拓扑管理
- `src/scene/shader_nodes.h` — 所有具体着色器节点类型（如 Diffuse BSDF、Principled BSDF 等）的定义
- `src/scene/scene.h` — `Scene` 类（`NodeOwner`），管理场景级节点的生命周期
- `src/util/param.h` — `ustring` 的定义，节点系统中所有名称使用的字符串类型
- `src/app/cycles_xml.cpp` — 使用 `node_xml.h` 接口进行场景 XML 文件的读写
