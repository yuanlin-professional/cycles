# types_float3.h - 三维浮点向量类型定义

## 概述

`types_float3.h` 定义了三维浮点向量结构体 `float3` 及其紧凑存储版本 `packed_float3`。`float3` 是 Cycles 中最核心的数学类型之一，广泛用于三维坐标、法线、颜色等场景。在 CPU 上，`float3` 通过 SSE `__m128` 实现 16 字节对齐以获得 SIMD 加速；在 GPU 上使用紧凑的 3 分量布局。`packed_float3` 为 12 字节紧凑存储格式，用于节省内存。

## 类与结构体

### `struct float3`

16 字节对齐（CPU）或紧凑布局（GPU）的三维浮点向量：

| 成员 | 类型 | 说明 |
|------|------|------|
| `x` | `float` | X 分量 |
| `y` | `float` | Y 分量 |
| `z` | `float` | Z 分量 |
| `w` | `float` | 填充分量（仅 CPU，用于 SIMD 对齐） |

**SSE 支持（CPU）**：
- 内部联合体包含 `__m128 m128` 成员
- 提供 `__m128` 的隐式/显式转换运算符
- 支持从 `__m128` 直接构造

**运算符重载（仅 CPU）**：
- `float operator[](int i) const` — 按索引只读访问
- `float &operator[](int i)` — 按索引读写访问

### `struct packed_float3`

12 字节紧凑存储版本，仅包含 `x`、`y`、`z` 三个 `float` 成员：

- 提供与 `float3` 的隐式互转
- 在 CUDA/HIP/oneAPI 上直接复用 `float3`（已经紧凑）
- Metal 使用原生 `packed_float3`
- 通过 `static_assert` 确保大小恰好为 12 字节

## 核心函数/运算符重载

| 函数签名 | 功能说明 |
|----------|----------|
| `float3 make_float3(float x, float y, float z)` | 从三个标量构造（SSE 使用 `_mm_set_ps`） |
| `float3 make_float3(float f)` | 广播单一标量到所有分量 |
| `float3 make_float3(float2 a)` | 从 `float2` 扩展（z=0） |
| `float3 make_float3(float2 a, float b)` | 从 `float2` + 标量组装 |
| `float3 make_float3(int3 i)` | 从 `int3` 转换（SSE 使用 `_mm_cvtepi32_ps`） |
| `float3 make_float3(float3 a)` | 恒等拷贝 |
| `float2 make_float2(float3 a)` | 截取前两个分量 |
| `int4 make_int4(float3 f)` | 转换为 `int4`（w 分量为 0） |
| `int3 make_int3(float3 f)` | 转换为 `int3` |
| `void print_float3(const char*, float3)` | 调试打印 |

## 依赖关系

- **内部头文件**:
  - `util/types_base.h`
  - `util/types_float2.h`
  - `util/types_int3.h`
  - `util/types_int4.h`
- **被引用**: `util/types_float4.h`、`util/types_spectrum.h`、`util/types_dual.h`、`util/types.h`、`util/math_float3.h`、`util/math_fast.h`、`kernel/osl/types.h`、`graph/node_type.cpp`

## 关联文件

- `src/util/math_float3.h` — `float3` 的数学运算（叉积、点积、归一化等）
- `src/util/types_spectrum.h` — 光谱类型（基于 `float3`）
- `src/util/types_float4.h` — 四维浮点向量（依赖本文件）
