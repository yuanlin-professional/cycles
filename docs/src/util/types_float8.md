# types_float8.h - 八维浮点向量类型定义（AVX 加速）

## 概述

`types_float8.h` 定义了八维浮点向量结构体 `vfloat8`（命名为 `vfloat8` 而非 `float8`，因为 `float8` 是 Metal 中的保留类型名）。该类型在 CPU 上通过 AVX `__m256` 提供 32 字节对齐的 SIMD 加速，主要用于 BVH 遍历中的 8 路并行射线-包围盒相交测试等高性能计算场景。原始实现源自 Intel，后被 Blender 基金会修改。

## 类与结构体

### `struct vfloat8`

32 字节对齐（CPU）或非对齐（GPU）的八维浮点向量：

| 成员 | 类型 | 说明 |
|------|------|------|
| `a` - `h` | `float` | 8 个浮点分量（a, b, c, d, e, f, g, h） |

**AVX 支持（CPU）**：
- 联合体包含 `__m256 m256` 成员
- 提供 `__m256` 隐式/显式转换
- 支持从 `__m256` 直接构造

**运算符重载（仅 CPU）**：
- `float operator[](int i) const` — 按索引只读访问（范围 0-7）
- `float &operator[](int i)` — 按索引读写访问

## 核心函数/运算符重载

| 函数签名 | 功能说明 |
|----------|----------|
| `vfloat8 make_vfloat8(float f)` | 广播单一标量到全部 8 个分量（AVX 使用 `_mm256_set1_ps`） |
| `vfloat8 make_vfloat8(float a, ..., float h)` | 从 8 个标量逐一构造 |
| `vfloat8 make_vfloat8(float4 a, float4 b)` | 从两个 `float4` 拼接（AVX 使用 `_mm256_insertf128_ps`） |
| `vint8 make_vint8(vfloat8 f)` | 转换为 `vint8`（AVX 使用 `_mm256_cvtps_epi32`） |
| `void print_vfloat8(const char*, vfloat8)` | 调试打印 8 个浮点分量 |

## 依赖关系

- **内部头文件**:
  - `util/types_base.h`
  - `util/types_float4.h`
  - `util/types_int8.h`
- **被引用**: `util/types.h`、`util/math_float8.h`、`util/math_int8.h`

## 关联文件

- `src/util/math_float8.h` — `vfloat8` 的数学运算
- `src/util/types_int8.h` — 对应的八维整数向量
- `src/util/types_float4.h` — 四维浮点向量（本文件的组成部分）
