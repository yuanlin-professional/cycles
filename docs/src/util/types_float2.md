# types_float2.h - 二维浮点向量类型定义

## 概述

`types_float2.h` 定义了二维浮点向量结构体 `float2`，包含 `x`、`y` 两个分量。该类型广泛用于 UV 坐标、二维屏幕坐标、纹理坐标等场景。文件提供了多种构造工厂函数 `make_float2` 以及与 `int2` 之间的类型转换函数。在 GPU 原生向量类型可用时，使用平台内置定义。

## 类与结构体

### `struct float2`

仅在非原生向量类型环境（`!__KERNEL_NATIVE_VECTOR_TYPES__`）下定义：

| 成员 | 类型 | 说明 |
|------|------|------|
| `x` | `float` | 第一分量 |
| `y` | `float` | 第二分量 |

**运算符重载（仅 CPU）**：
- `float operator[](int i) const` — 按索引只读访问（带断言边界检查）
- `float &operator[](int i)` — 按索引读写访问

## 核心函数/运算符重载

| 函数签名 | 功能说明 |
|----------|----------|
| `float2 make_float2(float x, float y)` | 从两个标量构造 `float2` |
| `float2 make_float2(float f)` | 广播单一标量到所有分量 |
| `float2 make_float2(int2 i)` | 从 `int2` 转换为 `float2` |
| `int2 make_int2(float2 f)` | 从 `float2` 截断转换为 `int2` |
| `void print_float2(const char*, float2)` | 调试打印，支持 Metal/CUDA/CPU |

## 依赖关系

- **内部头文件**:
  - `util/types_base.h`
  - `util/types_int2.h`
- **被引用**: `util/types_float3.h`、`util/types_dual.h`、`util/types.h`、`util/math_float2.h`

## 关联文件

- `src/util/math_float2.h` — `float2` 的数学运算（加减乘除、点积、长度等）
- `src/util/types_int2.h` — 对应的整数二维向量
- `src/util/types_float3.h` — 三维浮点向量（依赖本文件）
