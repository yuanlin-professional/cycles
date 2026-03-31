# node_util.h / node_util.cpp - Hydra渲染代理的节点参数类型转换工具

## 概述

本文件提供了通用场景描述(USD)的 `VtValue` 类型与 Cycles 节点 `SocketType` 之间的双向转换工具。`SetNodeValue` 和 `GetNodeValue` 是材质系统和渲染委托设置/读取 Cycles 节点参数的核心桥梁。文件内部实现了大量模板特化，覆盖了标量、向量、字符串、矩阵、枚举及其数组形式的完整转换链。

## 类与结构体

无独立类定义。

## 核心函数

### `SetNodeValue()`

- **签名**: `void SetNodeValue(Node *node, const SocketType &socket, const VtValue &value)`
- **功能**: 将 USD `VtValue` 转换为 Cycles 节点参数并设置到指定的 socket 上
- **支持的 SocketType**:
  - 标量: `BOOLEAN`, `FLOAT`, `INT`, `UINT`
  - 向量: `COLOR`, `VECTOR`, `POINT`, `NORMAL` -> `float3`; `POINT2` -> `float2`
  - 字符串: `STRING` -> `ustring`
  - 枚举: `ENUM` -> `ustring`（字符串输入）或 `int`（整数输入）
  - 变换: `TRANSFORM` -> `Transform`
  - 数组: `*_ARRAY` 版本对应上述所有类型
  - 特殊: `CLOSURE`（由连接处理，跳过）; `NODE` / `NODE_ARRAY`（未实现，输出警告）

### `GetNodeValue()`

- **签名**: `VtValue GetNodeValue(const Node *node, const SocketType &socket)`
- **功能**: 从 Cycles 节点读取参数值并转换为 USD `VtValue`
- **支持的类型**: 与 `SetNodeValue` 对称的完整读取链

## 依赖关系

- **内部头文件**:
  - `hydra/config.h` — 命名空间配置
  - `graph/node.h` — Cycles `Node`、`SocketType` 定义
  - `util/transform.h` — `Transform`、`make_transform()`
- **外部头文件**:
  - `pxr/base/vt/value.h`, `pxr/base/vt/array.h` — USD 值和数组类型
  - `pxr/base/gf/matrix3d.h`, `pxr/base/gf/matrix3f.h`, `pxr/base/gf/matrix4d.h`, `pxr/base/gf/matrix4f.h` — 矩阵类型
  - `pxr/base/gf/vec2f.h`, `pxr/base/gf/vec3f.h` — 向量类型
  - `pxr/usd/sdf/assetPath.h` — 资源路径类型
- **被引用**:
  - `hydra/material.cpp`, `hydra/render_delegate.cpp`

## 实现细节 / 关键算法

1. **模板转换框架**: 核心采用 `convertToCycles<DstType>(VtValue)` 模板函数，先尝试直接持有类型匹配，再尝试 `VtValue::Cast` 类型转换。对复杂类型（`float2`, `float3`, `ustring`, `Transform`）提供特化实现。

2. **float3 特化**: 支持从 `GfVec3f` 和 `GfVec4f`（丢弃 w 分量）两种输入转换为 `float3`。先尝试直接类型匹配，再尝试类型转换。

3. **ustring 特化**: 支持三种输入来源：`TfToken`、`std::string`、`SdfAssetPath`（使用已解析路径）。确保文件路径和标识符都能正确转换。

4. **矩阵转换**: `convertMatrixToCycles` 模板通过 SFINAE（`std::enable_if`）区分 3x3 和 4x4 矩阵，分别生成对应的 `Transform`。注意矩阵是转置存储的（行列互换）。

5. **数组转换**: `convertToCyclesArray<DstType, SrcType>` 处理数组类型。对于大小匹配的类型（`float`, `int`, `float4` 等）使用 `memcpy` 快速转换；对 `float3` 由于 padding 需要逐元素转换；对 `ustring` 和 `Transform` 等复杂类型使用自定义特化。

6. **反向转换**: `convertFromCycles` 和 `convertFromCyclesArray` 提供 Cycles -> USD 的反向转换，实现了 `GetNodeValue()` 的完整读取功能。

7. **枚举双模式**: `ENUM` 类型同时支持字符串（`TfToken`/`string` -> `ustring`）和整数输入，兼容 USD 中枚举值的不同表示方式。

## 关联文件

- `src/hydra/material.cpp` — 材质节点参数设置的主要使用者
- `src/hydra/render_delegate.cpp` — 渲染设置参数的读写
- `src/graph/node.h` — Cycles 节点和 socket 类型定义
