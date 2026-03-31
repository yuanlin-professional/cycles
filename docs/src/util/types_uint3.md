# types_uint3.h - 三维无符号整数向量类型定义

## 概述

`types_uint3.h` 定义了三维无符号整数向量结构体 `uint3` 及其紧凑存储版本 `packed_uint3`。`uint3` 用于三维无符号索引、GPU 线程网格维度等场景。`packed_uint3` 为 12 字节紧凑存储版本，解决了部分平台（如 HIP）的 `uint3` 内存对齐不紧凑的问题。

## 类与结构体

### `struct uint3`

仅在非原生向量类型环境下定义：

| 成员 | 类型 | 说明 |
|------|------|------|
| `x` | `uint` | X 分量 |
| `y` | `uint` | Y 分量 |
| `z` | `uint` | Z 分量 |

**运算符重载（仅 CPU）**：
- `uint operator[](uint i) const` — 按索引只读访问（带断言边界检查）
- `uint &operator[](uint i)` — 按索引读写访问

### `struct packed_uint3`

12 字节紧凑存储版本：

- 在 CUDA/oneAPI 上直接使用 `uint3`（通过 `using` 别名）
- Metal 使用原生 `packed_uint3`（注释中标注为 `packed_float3`，实际为无符号版本）
- HIP 及通用 CPU 平台提供独立结构体定义
- 提供与 `uint3` 的隐式互转及赋值运算符
- 通过 `static_assert` 确保大小恰好为 12 字节

## 核心函数/运算符重载

| 函数签名 | 功能说明 |
|----------|----------|
| `uint3 make_uint3(uint x, uint y, uint z)` | 从三个无符号整数构造 `uint3` |
| `packed_uint3 make_packed_uint3(uint x, uint y, uint z)` | 构造紧凑版 `packed_uint3` |

## 依赖关系

- **内部头文件**:
  - `util/types_base.h`
- **被引用**: `util/types.h`、`util/math_float3.h`

## 关联文件

- `src/util/types_base.h` — `uint` 类型别名定义
- `src/util/types.h` — 类型聚合头文件
- `src/util/math_float3.h` — 浮点三维数学运算（引用本文件）
