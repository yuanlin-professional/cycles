# types_float4.h - 四维浮点向量类型定义

## 概述

`types_float4.h` 定义了四维浮点向量结构体 `float4`，包含 `x`、`y`、`z`、`w` 四个分量，16 字节对齐。该类型广泛用于齐次坐标、RGBA 颜色、矩阵行向量等场景。在 CPU 上通过 SSE `__m128` 提供 SIMD 加速，在 GPU 上使用原生向量类型。文件还提供了与 `float3`、`int4` 之间的丰富转换函数。

## 类与结构体

### `struct float4`

16 字节对齐的四维浮点向量：

| 成员 | 类型 | 说明 |
|------|------|------|
| `x` | `float` | X 分量 |
| `y` | `float` | Y 分量 |
| `z` | `float` | Z 分量 |
| `w` | `float` | W 分量 |

**SSE 支持（CPU）**：
- 联合体包含 `__m128 m128` 成员
- 提供 `__m128` 隐式/显式转换
- 支持从 `__m128` 直接构造

**运算符重载（仅 CPU）**：
- `float operator[](int i) const` — 按索引只读访问（带断言检查 0-3）
- `float &operator[](int i)` — 按索引读写访问

## 核心函数/运算符重载

| 函数签名 | 功能说明 |
|----------|----------|
| `float4 make_float4(float x, float y, float z, float w)` | 从四个标量构造（SSE 使用 `_mm_set_ps`） |
| `float4 make_float4(float f)` | 广播单一标量到所有分量 |
| `float4 make_float4(float3 a, float b)` | 从 `float3` + w 分量组装 |
| `float4 make_float4(float3 a)` | 从 `float3` 扩展（w=1.0） |
| `float4 make_homogeneous(float3 a)` | 构造齐次坐标（w=1.0） |
| `float4 make_float4(int4 i)` | 从 `int4` 转换（SSE 使用 `_mm_cvtepi32_ps`） |
| `float3 make_float3(float4 a)` | 截取前三个分量 |
| `int4 make_int4(float4 f)` | 转换为 `int4` |
| `void print_float4(const char*, float4)` | 调试打印，支持 Metal/CUDA/CPU |

## 依赖关系

- **内部头文件**:
  - `util/types_base.h`
  - `util/types_float3.h`
  - `util/types_int4.h`
- **被引用**: `util/types_float8.h`、`util/types_dual.h`、`util/types.h`、`util/math_float4.h`、`util/math_float3.h`、`util/math_float2.h`、`util/math_int4.h`、`util/math_fast.h`、`util/string.cpp`

## 关联文件

- `src/util/math_float4.h` — `float4` 的数学运算（点积、混合、比较等）
- `src/util/types_float3.h` — 三维浮点向量
- `src/util/types_float8.h` — 八维浮点向量（依赖本文件）
