# attribute.h / attribute.cpp - Hydra渲染代理的图元变量属性转换工具

## 概述

本文件提供了将 Hydra 图元变量（Primvar）数据转换并写入 Cycles 属性集（AttributeSet）的工具函数。`ApplyPrimvars` 是几何图元（网格、曲线、点云）在同步过程中将通用场景描述(USD)的顶点/面/实例属性数据转换为 Cycles 内部 `Attribute` 的核心桥梁。该函数处理多种数据类型的转换，特别是 `float3` 的 padding 问题。

## 类与结构体

无独立类定义。

## 核心函数

### `ApplyPrimvars()`

- **签名**: `void ApplyPrimvars(AttributeSet &attributes, const ustring &name, VtValue value, AttributeElement elem, AttributeStandard std)`
- **参数**:
  - `attributes` — Cycles 属性集，属性将被添加到此集合
  - `name` — 属性名称（如 `"st"`、`"N"` 等）
  - `value` — Hydra 传入的 `VtValue` 数据
  - `elem` — 属性元素类型（`ATTR_ELEMENT_VERTEX`、`ATTR_ELEMENT_FACE` 等）
  - `std` — 属性标准类型（`ATTR_STD_UV`、`ATTR_STD_NONE` 等）
- **功能**: 根据 `HdType` 确定 Cycles 属性类型，执行数据转换并拷贝到属性缓冲区
- **支持的类型映射**:
  - `HdTypeFloat` -> `TypeFloat`（`sizeof(float)`）
  - `HdTypeFloatVec2` -> `TypeFloat2`（`sizeof(float2)`）
  - `HdTypeFloatVec3` -> `TypeVector`（`sizeof(float3)`，需要从 `GfVec3f` 转换）
  - `HdTypeFloatVec4` -> `TypeFloat4`（`sizeof(float4)`）

## 依赖关系

- **内部头文件**:
  - `hydra/config.h` — 命名空间配置
  - `scene/attribute.h` — Cycles `AttributeSet`、`Attribute`、`AttributeElement`、`AttributeStandard`
- **外部头文件**:
  - `pxr/base/vt/value.h` — USD 通用值类型
  - `pxr/base/vt/array.h` — USD 数组类型
  - `pxr/imaging/hd/types.h` — `HdType`、`HdGetValueData()`、`HdGetValueTupleType()`
  - `pxr/base/gf/vec2f.h`, `pxr/base/gf/vec3f.h`, `pxr/base/gf/vec4f.h` — USD 向量类型
  - `pxr/imaging/hd/tokens.h` — Hydra token
- **被引用**:
  - `hydra/mesh.cpp`, `hydra/curves.cpp`, `hydra/pointcloud.cpp`（所有几何图元）

## 实现细节 / 关键算法

1. **float3 Padding 转换**: Cycles 的 `float3` 实际存储为 `float4`（16 字节对齐），而 USD 的 `GfVec3f` 是紧凑的 12 字节。因此 `HdTypeFloatVec3` 需要逐元素转换（`make_float3(vec[0], vec[1], vec[2])`），而不能简单地内存拷贝。转换后的数据存储在局部 `VtArray<float3>` 中，原始 `value` 变量被覆盖以维持数据生命周期。

2. **编译期大小验证**: 使用 `static_assert(sizeof(GfVec2f) == sizeof(float2))` 和 `static_assert(sizeof(GfVec4f) == sizeof(float4))` 确保 2 分量和 4 分量类型可以安全地进行内存拷贝。

3. **数据大小计算**: 通过 `value.GetArraySize()` 获取元素数量，再乘以目标类型的大小得到字节数。最终通过 `assert(size == attr->buffer.size())` 验证计算的大小与属性缓冲区大小一致。

4. **零拷贝路径**: 对于 `float`、`float2`、`float4` 类型，直接使用 `HdGetValueData()` 返回的原始数据指针进行 `memcpy`，无需额外转换。

## 关联文件

- `src/hydra/mesh.cpp` — 网格图元同步顶点属性
- `src/hydra/curves.cpp` — 曲线图元同步控制点属性
- `src/hydra/pointcloud.cpp` — 点云图元同步点属性
- `src/scene/attribute.h` — Cycles 属性系统定义
