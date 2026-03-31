# node_xml.h / node_xml.cpp - 节点的 XML 序列化与反序列化

## 概述

本文件实现了 Cycles 渲染器中节点 (节点) 与 XML 格式之间的双向序列化功能。通过 pugixml 库，`xml_read_node()` 可将 XML 元素中的属性值解析并写入节点的各个 socket，`xml_write_node()` 则将节点的所有非默认 socket 值导出为 XML 元素和属性。该模块是 Cycles 独立运行（`cycles_xml` 应用）中场景文件加载/保存的关键组件，整个文件受 `WITH_PUGIXML` 编译宏保护。

## 类与结构体

### XMLReader
- **功能**: XML 读取上下文，维护一个名称到节点指针的映射表，用于在反序列化过程中解析节点间的引用关系。
- **关键成员**:
  - `node_map` (`map<ustring, Node *>`) — 已读取节点的名称映射。当 XML 中某个 NODE 类型 socket 的值为节点名称时，通过此映射解析实际的节点指针。

## 核心函数

### xml_read_node()
- **签名**: `void xml_read_node(XMLReader &reader, Node *node, const xml_node xml_node)`
- **功能**: 从 XML 节点中读取属性值并设置到目标节点的对应 socket 中。处理流程：
  1. 读取 `name` 属性设置节点名称。
  2. 遍历目标节点类型的所有输入 socket，跳过 CLOSURE、UNDEFINED 和 INTERNAL 类型。
  3. 对每个 socket，查找同名 XML 属性，若存在则根据 socket 类型进行解析和设置。
  4. 读取完成后将节点注册到 `reader.node_map` 中。

  支持的类型解析：
  - `BOOLEAN` — 通过 `xml_read_boolean()` 解析 "true"/"false" 或数值。
  - `FLOAT / INT / UINT / UINT64` — 通过 `atof()` / `atoi()` / `strtoull()` 解析。
  - `COLOR / VECTOR / POINT / NORMAL` — 使用 `xml_read_float_array<3>()` 解析 3 分量浮点数组。
  - `POINT2` — 使用 `xml_read_float_array<2>()` 解析 2 分量浮点数组。
  - `TRANSFORM` — 使用 `xml_read_float_array<12>()` 解析 4x3 变换矩阵的 12 个浮点数。
  - `STRING` — 直接设置字符串值。
  - `ENUM` — 按名称查找枚举值，不存在时输出错误日志。
  - `NODE` — 从 `reader.node_map` 中按名称查找引用节点，并验证类型兼容性。
  - `NODE_ARRAY` — 空格分隔的多个节点名称引用。
  - 各类型的 `_ARRAY` 版本 — 空格分隔的多值解析。

### xml_write_node()
- **签名**: `xml_node xml_write_node(Node *node, xml_node xml_root)`
- **功能**: 将节点的非默认 socket 值写入 XML。处理流程：
  1. 在 `xml_root` 下创建以节点类型名称命名的子元素。
  2. 设置 `name` 属性。
  3. 遍历所有输入 socket，跳过 CLOSURE、UNDEFINED、INTERNAL 类型以及仍为默认值的 socket。
  4. 根据 socket 类型将值格式化为字符串并设置为 XML 属性。

  格式化规则：
  - `BOOLEAN` — 输出 "true" 或 "false"。
  - `FLOAT` — 转为 double 设置。
  - `COLOR / VECTOR / POINT / NORMAL` — 格式化为 `"%g %g %g"` 空格分隔的三元组。
  - `POINT2` — 格式化为 `"%g %g"` 空格分隔的二元组。
  - `TRANSFORM` — 输出 4x4 矩阵的 16 个浮点数（最后一行固定为 `0 0 0 1`）。
  - `STRING / ENUM` — 直接输出字符串值。
  - `NODE` — 输出被引用节点的名称。
  - 数组类型 — 空格分隔的多值输出。

### xml_read_boolean() (静态)
- **签名**: `static bool xml_read_boolean(const char *value)`
- **功能**: 将字符串解析为布尔值，支持 "true"（大小写不敏感）或非零整数。

### xml_read_float_array<VECTOR_SIZE>() (静态模板)
- **签名**: `template<int VECTOR_SIZE, typename T> static void xml_read_float_array(T &value, xml_attribute attr)`
- **功能**: 将空格分隔的浮点数字符串解析为 `VECTOR_SIZE` 分量的向量数组。验证 token 数量是否为 `VECTOR_SIZE` 的整数倍。

## 依赖关系

- **内部头文件**:
  - `graph/node_xml.h` — 自身头文件
  - `graph/node.h` — Node 基类（.cpp）
  - `util/map.h` — map 容器
  - `util/param.h` — ustring
  - `util/xml.h` — pugixml 的封装（xml_node、xml_attribute 类型）
  - `util/log.h`（.cpp）— 错误日志
  - `util/string.h`（.cpp）— string_split()、string_iequals()、string_printf()
  - `util/transform.h`（.cpp）— Transform 类型
- **外部库**: pugixml（通过 `WITH_PUGIXML` 编译开关控制）
- **被引用**:
  - `src/app/cycles_xml.cpp` — Cycles 独立 XML 场景加载器

## 实现细节 / 关键算法

1. **编译条件保护**: 整个头文件和实现文件均被 `#ifdef WITH_PUGIXML` / `#endif` 包围。当未链接 pugixml 库时，相关代码完全不参与编译。

2. **节点引用解析机制**: XML 中的 NODE 和 NODE_ARRAY 类型 socket 通过节点名称进行引用。`XMLReader::node_map` 在读取过程中逐步构建：每个节点读取完毕后将自身注册到映射表中。因此 XML 文件中的节点定义顺序必须保证被引用节点在引用节点之前出现（拓扑排序）。

3. **类型安全的节点引用**: 在解析 NODE 类型引用时，不仅查找名称映射，还通过 `value_node->is_a(socket.node_type)` 验证被引用节点的类型是否兼容，不兼容的引用会被静默忽略。

4. **默认值跳过**: `xml_write_node()` 在输出前通过 `node->has_default_value(socket)` 检查，跳过仍为默认值的 socket，从而产生更紧凑的 XML 输出。

5. **Transform 序列化**: Transform 是 3x4 矩阵（内部 3 行 4 列），写入 XML 时输出完整的 4x4 矩阵（补齐最后一行 `0 0 0 1`），读取时以 12 个浮点数为一组解析。

## 关联文件

- `src/graph/node.h` / `src/graph/node.cpp` — 节点基类，提供 socket 值的读写接口
- `src/graph/node_type.h` — SocketType 定义，XML 序列化依据 socket 类型分派
- `src/app/cycles_xml.cpp` — 使用本模块加载 XML 格式的场景文件
- `src/util/xml.h` — pugixml 库的封装头文件
