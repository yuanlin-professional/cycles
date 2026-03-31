# types_int3.h - 三维整数向量类型定义

## 概述

`types_int3.h` 定义了三维整数向量结构体 `int3` 及其紧凑存储版本 `packed_int3`。`int3` 在 CPU 上通过 SSE `__m128i` 实现 16 字节对齐以支持 SIMD 运算，在 GPU 上使用紧凑的 3 分量布局。`packed_int3` 为 12 字节紧凑存储格式，适合内存敏感场景。该类型用于三维网格索引、体素坐标等整数空间的三维量。

## 类与结构体

### `struct int3`

16 字节对齐（CPU）或紧凑布局（GPU）的三维整数向量：

| 成员 | 类型 | 说明 |
|------|------|------|
| `x` | `int` | X 分量 |
| `y` | `int` | Y 分量 |
| `z` | `int` | Z 分量 |
| `w` | `int` | 填充分量（仅 CPU，用于 SIMD 对齐） |

**SSE 支持（CPU）**：
- 联合体包含 `__m128i m128` 成员
- 提供 `__m128i` 隐式/显式转换
- 支持从 `__m128i` 直接构造

### `struct packed_int3`

12 字节紧凑存储版本（仅在 HIP 及通用 CPU 平台定义）：

- 在 CUDA/oneAPI 上，`int3` 本身已经紧凑，直接使用 `typedef`
- Metal 使用原生 `packed_int3`
- 提供与 `int3` 的隐式互转及赋值运算符
- 通过 `static_assert` 确保大小恰好为 12 字节

## 核心函数/运算符重载

| 函数签名 | 功能说明 |
|----------|----------|
| `int3 make_int3(int x, int y, int z)` | 从三个标量构造（SSE 使用 `_mm_set_epi32`） |
| `int3 make_int3(int i)` | 广播单一标量到所有分量 |
| `packed_int3 make_packed_int3(int x, int y, int z)` | 构造紧凑版 `packed_int3` |
| `void print_int3(const char*, int3)` | 调试打印 |

## 依赖关系

- **内部头文件**:
  - `util/types_base.h`
- **被引用**: `util/types_float3.h`、`util/types.h`、`util/math_int3.h`、`util/math_float3.h`、`kernel/util/nanovdb.h`

## 关联文件

- `src/util/math_int3.h` — `int3` 的数学运算
- `src/util/types_float3.h` — 对应的浮点三维向量（引用本文件进行类型转换）
