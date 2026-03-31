# types_int4.h - 四维整数向量类型定义

## 概述

`types_int4.h` 定义了四维整数向量结构体 `int4`，包含 `x`、`y`、`z`、`w` 四个 `int` 分量，16 字节对齐。在 CPU 上通过 SSE `__m128i` 提供 SIMD 加速。该类型用于 BVH 节点索引、像素四元组操作等场景。文件还提供了零值工厂函数 `zero_int4()`。

## 类与结构体

### `struct int4`

16 字节对齐的四维整数向量：

| 成员 | 类型 | 说明 |
|------|------|------|
| `x` | `int` | X 分量 |
| `y` | `int` | Y 分量 |
| `z` | `int` | Z 分量 |
| `w` | `int` | W 分量 |

**SSE 支持（CPU）**：
- 联合体包含 `__m128i m128` 成员
- 提供 `__m128i` 隐式/显式转换
- 支持从 `__m128i` 直接构造

**运算符重载（仅 CPU）**：
- `int operator[](int i) const` — 按索引只读访问（范围 0-3）
- `int &operator[](int i)` — 按索引读写访问

## 核心函数/运算符重载

| 函数签名 | 功能说明 |
|----------|----------|
| `int4 make_int4(int x, int y, int z, int w)` | 从四个标量构造（SSE 使用 `_mm_set_epi32`） |
| `int4 make_int4(int i)` | 广播单一标量到所有分量 |
| `int4 zero_int4()` | 返回全零 `int4` |
| `void print_int4(const char*, int4)` | 调试打印 |

## 依赖关系

- **内部头文件**:
  - `util/types_base.h`
- **被引用**: `util/types_int8.h`、`util/types_float3.h`、`util/types_float4.h`、`util/types.h`、`util/rect.h`、`util/math_int4.h`

## 关联文件

- `src/util/math_int4.h` — `int4` 的数学运算与位操作
- `src/util/types_float4.h` — 对应的浮点四维向量（引用本文件进行类型转换）
- `src/util/types_int8.h` — 八维整数向量（依赖本文件）
- `src/util/rect.h` — 矩形区域定义（使用 `int4`）
