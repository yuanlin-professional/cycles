# math_int4.h - 四维整数向量运算

## 概述

`math_int4.h` 为 `int4` 类型提供完整的数学运算支持，包括算术、位运算、比较、移位以及 SIMD shuffle 操作。int4 在 Cycles 中广泛用于矩形区域表示（见 `rect.h`）、层次包围体(BVH) 遍历中的索引存储、以及作为 float4 比较运算的掩码类型。

## 核心函数

### 运算符重载（SSE 优化）
| 运算符 | 说明 |
|--------|------|
| `+` / `+=` / `-` / `-=` | 加减法（SSE: `_mm_add/sub_epi32`） |
| `*` | 乘法（标量回退） |
| `>>` / `<<` | 右移 / 左移（SSE: `_mm_srai/slli_epi32`） |
| `<` / `==` / `>=` | 比较运算（SSE: `_mm_cmplt/cmpeq_epi32`） |
| `&` / `\|` / `^` | 按位与/或/异或（SSE: `_mm_and/or/xor_si128`） |
| `&=` / `\|=` / `^=` / `<<=` / `>>=` | 复合赋值版本 |

### 数学函数
| 函数 | 说明 |
|------|------|
| `min(a, b)` / `max(a, b)` | 逐分量最小/最大值（SSE4.2: `_mm_min/max_epi32`） |
| `clamp(a, mn, mx)` | 逐分量截断 |
| `select(mask, a, b)` | 条件选择（SSE: `_mm_or_si128 + _mm_and/andnot_si128`） |
| `cast(int4)` → `float4` | 位级别重解释 |
| `load_int4(int*)` | 从内存加载（SSE: `_mm_loadu_si128`） |

### SSE SIMD 工具
| 函数 | 说明 |
|------|------|
| `srl(a, b)` | 逻辑右移（`_mm_srli_epi32`，不符号扩展） |
| `andnot(a, b)` | `~a & b`（`_mm_andnot_si128`） |
| `shuffle<i0,i1,i2,i3>(a)` | 分量重排（SSE: `_mm_shuffle_epi32`；Neon 支持） |
| `shuffle<i0,i1,i2,i3>(a, b)` | 两向量交叉重排 |

## 依赖关系

- **内部头文件**: `util/types_float4.h`, `util/types_int4.h`
- **被引用**: 通过 `util/math.h` 间接引用；直接被 `math_fast.h`, `math_float3.h`, `rect.h` 引用；在 BVH 遍历和矩形区域运算中大量使用

## 实现细节 / 关键算法

1. **`>=` 运算符**: SSE 缺少 `cmpge` 整数指令，因此通过 `xor(0xffffffff, cmpgt(b, a))` 即"非(b>a)"来实现。

2. **`srl` 逻辑右移**: 与算术右移 `>>` (`_mm_srai_epi32`) 不同，`srl` 使用 `_mm_srli_epi32` 执行无符号/逻辑右移，高位补零而非符号扩展。这在处理无符号索引时很重要。

3. **Neon shuffle**: ARM 平台通过 `shuffle_neon<int32x4_t>` 辅助模板函数将 `_MM_SHUFFLE` 模式翻译为 Neon 的向量重排指令。

## 关联文件

- `util/types_int4.h` — int4 结构体定义（含 `__m128i m128` 成员）
- `util/rect.h` — 矩形区域使用 int4 存储 (x0, y0, x1, y1)
- `util/math_float4.h` — float4/int4 之间的 cast 操作
