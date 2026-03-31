# node_enum.h - 节点枚举值双向映射工具

## 概述

本文件定义了 `NodeEnum` 结构体，提供枚举字符串名称与整数值之间的双向映射功能。它是 Cycles 渲染器节点 (节点) 类型系统中处理枚举属性的核心工具类。在节点的 ENUM 类型 socket 中，`NodeEnum` 被用来在用户可读的字符串标识符和内部整数表示之间进行转换，广泛应用于着色器节点、场景设置等所有包含枚举选项的属性定义中。

## 类与结构体

### NodeEnum
- **功能**: 维护 `ustring <-> int` 的双向映射，支持通过字符串查找整数值或通过整数查找字符串名称。同时支持迭代器接口以遍历所有已注册的枚举项。
- **关键成员**:
  - `left` (`unordered_map<ustring, int>`, private) — 字符串到整数的映射表。
  - `right` (`unordered_map<int, ustring>`, private) — 整数到字符串的反向映射表。

- **关键方法**:
  - `empty()` — 检查枚举表是否为空。
  - `insert(const char *x, int y)` — 插入一对枚举映射（字符串名称 -> 整数值），同时更新双向映射。内部将 `const char*` 转换为 `ustring` 以获得更高效的哈希查找。
  - `exists(ustring x)` — 检查指定字符串名称是否存在于枚举中。
  - `exists(int y)` — 检查指定整数值是否存在于枚举中。
  - `operator[](const char *x)` — 通过 C 字符串名称查找对应的整数值。
  - `operator[](ustring x)` — 通过 ustring 名称查找对应的整数值。
  - `operator[](int y)` — 通过整数值查找对应的字符串名称。
  - `begin() / end()` — 返回字符串到整数映射（`left`）的 const 迭代器，支持范围遍历。

## 枚举与常量

本文件本身不定义枚举常量，但 `NodeEnum` 的实例在整个 Cycles 代码库中被用来注册各种枚举类型（如插值模式、着色器类型、纹理映射方式等）。

## 依赖关系

- **内部头文件**:
  - `util/map.h` — `unordered_map` 容器
  - `util/param.h` — `ustring` 类型
- **外部库**: 无
- **被引用**:
  - `src/graph/node_type.h` — SocketType 的 `enum_values` 成员使用 `const NodeEnum *`

## 实现细节 / 关键算法

1. **双向映射设计**: 通过同时维护 `left`（string -> int）和 `right`（int -> string）两个哈希表实现 O(1) 复杂度的双向查找。`insert()` 方法同时更新两个映射表以保持一致性。

2. **ustring 优化**: 字符串键使用 `ustring`（唯一字符串/内联字符串），该类型在底层进行字符串内化（string interning），使得字符串比较退化为指针比较，大幅提升哈希查找性能。

3. **仅头文件实现**: `NodeEnum` 的所有方法均在头文件中内联定义，无对应的 `.cpp` 实现文件。这简化了编译依赖并便于内联优化。

4. **迭代器接口**: `begin()` 和 `end()` 暴露了 `left` 映射的迭代器，允许外部代码遍历所有枚举项（如在 XML 序列化中列出所有合法枚举值，或在 UI 中生成下拉选项列表）。

## 关联文件

- `src/graph/node_type.h` / `src/graph/node_type.cpp` — 在 SocketType 中引用 NodeEnum 作为枚举值定义
- `src/graph/node.cpp` — Node::set(ustring) 中通过 NodeEnum 将字符串转换为枚举整数值
- `src/graph/node_xml.cpp` — XML 反序列化时通过 NodeEnum 验证枚举值合法性
