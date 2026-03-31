# math_int8.h - 八维整数向量运算（AVX）

## 概述

`math_int8.h` 为 `vint8` 类型提供数学运算支持。vint8 是 8 路宽度的整数 SIMD 向量，在 AVX 平台上映射到 `__m256i` 寄存器。它是 `vfloat8` 的整数对应类型，用于光谱渲染中的掩码运算、索引操作以及类型转换的中间类型。仅在 CPU (`__KERNEL_GPU__` 未定义) 上可用。

## 核心函数

### 运算符重载（AVX 优化）
| 运算符 | 说明 |
|--------|------|
| `+` / `+=` / `-` / `-=` | 加减法（AVX: `_mm256_add/sub_epi32`） |
| `>>` / `<<` | 移位（AVX: `_mm256_srai/slli_epi32`） |
| `<` / `==` / `>=` | 比较（AVX: `_mm256_cmpgt/cmpeq_epi32`） |
| `&` / `\|` / `^` | 位运算（AVX: `_mm256_and/or/xor_si256`） |
| 标量版本 | 所有位运算和移位都有与 `int32_t` 标量交互的重载 |
| 复合赋值 | `&=`, `\|=`, `^=`, `<<=`, `>>=` |

### 数学函数
| 函数 | 说明 |
|------|------|
| `min(a, b)` / `max(a, b)` | 逐分量最小/最大值（AVX4.1: `_mm256_min/max_epi32`） |
| `clamp(a, mn, mx)` | 逐分量截断 |
| `select(mask, a, b)` | 条件选择（AVX: `_mm256_blendv_ps`通过类型转换） |
| `load_vint8(int*)` | 从内存加载（AVX: `_mm256_loadu_si256`） |
| `cast(vint8)` → `vfloat8` | 位级别重解释为 vfloat8 |

### SSE/AVX SIMD 工具（仅 `__KERNEL_AVX__`）
| 函数 | 说明 |
|------|------|
| `srl(a, b)` | 逻辑右移（`_mm256_srli_epi32`） |
| `shuffle<i>(a)` | 单索引广播 |
| `shuffle<i0,i1>(a)` / `shuffle<i0,i1>(a, b)` | 128 位 lane 级别重排（`_mm256_permute2f128_si256`） |
| `shuffle<i0,i1,i2,i3>(a)` / `shuffle<i0,i1,i2,i3>(a, b)` | lane 内分量重排 |

## 依赖关系

- **内部头文件**: `util/math_base.h`, `util/types_float8.h`, `util/types_int8.h`
- **被引用**: 通过 `util/math.h` 间接引用；用作 `vfloat8` 条件选择的掩码类型

## 实现细节 / 关键算法

1. **`>=` 运算符**: 与 int4 类似，通过 `xor(全1, cmpgt(b, a))` 实现，因为 AVX2 没有直接的 `cmpge` 整数指令。

2. **`select`**: 使用 `_mm256_blendv_ps` 实现，需要在 int 和 float 之间进行类型转换（`_mm256_castsi256_ps` / `_mm256_castps_si256`），因为 AVX 的 blendv 只有浮点版本。

3. **shuffle 的多层次模板**: 提供了 1-index（广播）、2-index（128 位 lane 交换）和 4-index（lane 内重排）三种粒度。常用模式如 `<0,0,2,2>`, `<1,1,3,3>`, `<0,1,0,1>` 有使用 `moveldup`/`movehdup`/`movedup` 的特化版本。

4. **CPU 限定**: 整个文件（运算符部分）被 `#ifndef __KERNEL_GPU__` 包裹，因为 GPU 后端不使用 vint8 类型。

## 关联文件

- `util/types_int8.h` — vint8 结构体定义（含 `__m256i m256` 成员）
- `util/math_float8.h` — 配套的 vfloat8 浮点运算
- `util/math_int4.h` — 结构上类似的 128 位版本
