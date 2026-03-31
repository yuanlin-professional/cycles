# math_int3.h - 三维整数向量运算

## 概述

`math_int3.h` 为 `int3` 类型提供数学运算支持，包括算术运算、比较运算、位运算以及 min/max/clamp 等操作。int3 在 Cycles 中用于体素网格索引、三角形索引三元组以及条件掩码向量（作为 float3 比较运算的返回类型）。

## 核心函数

### 运算符重载
| 运算符 | 说明 |
|--------|------|
| `+` / `-` | 加法 / 减法（SSE: `_mm_add/sub_epi32`） |
| `*` | 乘法 |
| `>>` | 右移（标量） |
| `&` | 按位与（SSE: `_mm_and_si128`） |
| `^` | 按位异或 |
| `==` / `!=` / `<` | 比较运算 |

### 数学函数
| 函数 | 说明 |
|------|------|
| `min(a, b)` / `max(a, b)` | 逐分量最小/最大值（SSE4.2: `_mm_min/max_epi32`） |
| `clamp(a, mn, mx)` | 逐分量截断（两种重载：标量边界和 int3 边界） |
| `all(a)` | 所有分量非零则返回 true |

## 依赖关系

- **内部头文件**: `util/types_int3.h`
- **被引用**: 通过 `util/math.h` 间接被引用；用于 float3 比较返回的掩码类型、体素网格索引

## 实现细节 / 关键算法

- SSE4.2 下的 `min`/`max` 使用 `_mm_min_epi32`/`_mm_max_epi32` 整数 SIMD 指令
- SSE 下的 `+`/`-` 使用 `_mm_add_epi32`/`_mm_sub_epi32`
- `&` 运算符有两个重载：int3 与 int3 之间，以及 int3 与标量 int 之间
- Metal 平台下这些运算符被跳过，因为 Metal 内建支持

## 关联文件

- `util/types_int3.h` — int3 结构体定义（含 `__m128i m128` 成员）
- `util/math_float3.h` — float3 的比较运算返回 int3 掩码
