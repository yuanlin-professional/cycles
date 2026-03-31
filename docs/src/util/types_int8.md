# types_int8.h - 八维整数向量类型定义（AVX 加速）

## 概述

`types_int8.h` 定义了八维整数向量结构体 `vint8`，在 CPU 上通过 AVX `__m256i` 提供 32 字节对齐的 SIMD 加速。该类型与 `vfloat8` 配合使用，主要用于 BVH 8 路并行遍历中的索引操作和掩码计算。在非 AVX 平台上回退为 8 个独立 `int` 成员。

## 类与结构体

### `struct vint8`

32 字节对齐（CPU）或非对齐（GPU）的八维整数向量：

| 成员 | 类型 | 说明 |
|------|------|------|
| `a` - `h` | `int` | 8 个整数分量（a, b, c, d, e, f, g, h） |

**AVX 支持（CPU）**：
- 联合体包含 `__m256i m256` 成员
- 提供 `__m256i` 隐式/显式转换
- 支持从 `__m256i` 直接构造

**运算符重载（仅 CPU）**：
- `int operator[](int i) const` — 按索引只读访问（范围 0-7）
- `int &operator[](int i)` — 按索引读写访问

**前向声明**：文件中前向声明了 `struct vfloat8`，以避免循环依赖。

## 核心函数/运算符重载

| 函数签名 | 功能说明 |
|----------|----------|
| `vint8 make_vint8(int a, ..., int h)` | 从 8 个标量逐一构造（AVX 使用 `_mm256_set_epi32`） |
| `vint8 make_vint8(int i)` | 广播单一标量到全部 8 个分量 |
| `vint8 make_vint8(int4 a, int4 b)` | 从两个 `int4` 拼接（AVX 使用 `_mm256_insertf128_si256`） |

## 依赖关系

- **内部头文件**:
  - `util/types_base.h`
  - `util/types_int4.h`
- **被引用**: `util/types_float8.h`、`util/types.h`、`util/math_int8.h`、`util/math_float8.h`

## 关联文件

- `src/util/math_int8.h` — `vint8` 的数学运算
- `src/util/types_float8.h` — 对应的八维浮点向量
- `src/util/types_int4.h` — 四维整数向量（本文件的组成部分）
