# math_float4.h - 四维浮点向量运算

## 概述

`math_float4.h` 为 `float4` 类型提供完整的数学运算支持。float4 是 Cycles 中 SIMD 运算的核心载体类型，在 CPU 上直接映射到 `__m128` 寄存器。该文件在渲染器中用于颜色(RGBA)、齐次坐标、SIMD 批量处理等场景，并为 `float3` 的 SSE 实现提供底层 shuffle 等关键操作。

## 核心函数

### 构造与初始化
| 函数 | 说明 |
|------|------|
| `zero_float4()` | 零向量（SSE: `_mm_setzero_ps`） |
| `one_float4()` | 全 1 向量 |
| `cast(float4)` → `int4` | 位级别重解释为 int4 |

### 运算符重载（SSE 优化）
- 算术: `+`, `-`, `*`, `/`, `^` (XOR)
- 复合赋值: `+=`, `-=`, `*=`, `/=`
- 比较: `<`, `>=`, `<=`, `==`

### SIMD 专用操作（`__KERNEL_SSE__`）
| 函数 | 说明 |
|------|------|
| `shuffle<i0,i1,i2,i3>(a)` | 分量重排（`_mm_shuffle_epi32`） |
| `shuffle<i0,i1,i2,i3>(a, b)` | 两向量交叉重排（`_mm_shuffle_ps`） |
| `extract<i>(a)` | 提取第 i 个分量 |
| `madd(a, b, c)` | 融合乘加 `a*b+c`（AVX2 下使用 `_mm_fmadd_ps`，Neon 使用 `vfmaq_f32`） |
| `msub(a, b, c)` | 融合乘减 `a*b-c` |
| `load_float4(float*)` | 从内存加载（`_mm_loadu_ps`，非对齐） |

### 向量数学
| 函数 | 说明 |
|------|------|
| `dot(a, b)` | 点积（SSE4.2: `_mm_dp_ps` 全四分量） |
| `cross(a, b)` | 叉积（SSE: shuffle 组合） |
| `len(a)` / `len_squared(a)` | 长度 / 长度平方 |
| `normalize(a)` / `safe_normalize(a)` | 归一化 |
| `distance(a, b)` | 两点距离 |

### 分量操作
| 函数 | 说明 |
|------|------|
| `reduce_add(a)` | 分量求和（SSE3: `_mm_hadd_ps`；Neon: `vaddvq_f32`） |
| `reduce_min(a)` / `reduce_max(a)` | 分量极值 |
| `average(a)` | 分量均值 `sum/4` |
| `min(a,b)` / `max(a,b)` / `clamp` | 逐分量 min/max/clamp |
| `fabs` / `fmod` / `sqrt` / `floor` | 逐分量数学函数 |
| `floorfrac(x, *i)` | 同时获取整数部分和小数部分 |
| `mix` / `interp` / `saturate` | 插值 |
| `exp` / `log` / `power` | 逐分量超越函数 |
| `safe_divide` / `is_zero` / `isfinite_safe` / `ensure_finite` | 安全操作 |
| `select(mask, a, b)` / `mask(mask, a)` | 条件选择（SSE4.2: `blendv`） |

### 类型转换
| 函数 | 说明 |
|------|------|
| `__float4_as_int4` / `__int4_as_float4` | 位级别重解释 |
| `copy_v4_v4(float*, float4)` | 复制到原始 float 数组 |

## 依赖关系

- **内部头文件**: `util/math_base.h`, `util/types_float4.h`
- **被引用**: 通过 `util/math.h` 被所有模块引用；直接被 `math_float3.h`（SSE 叉积/归一化依赖 float4 shuffle）、`math_fast.h`（SIMD 版快速数学）、`math_intersect.h` 引用

## 实现细节 / 关键算法

1. **`madd` / `msub`**: 在 AVX2 CPU 上编译为单条 `vfmaddps`/`vfmsubps` 指令，在 ARM Neon 上编译为 `vfmaq_f32`，其他平台回退到两条独立指令。这是快速多项式求值（如 `math_fast.h`）的关键基础设施。

2. **`shuffle` 模板特化**: 针对常用重排模式（如 `<0,1,0,1>`, `<2,3,2,3>`）使用 `movelh_ps`/`movehl_ps` 等更高效的指令替代通用 `_mm_shuffle_ps`。SSE3 下 `<0,0,2,2>` 和 `<1,1,3,3>` 使用 `moveldup_ps`/`movehdup_ps`。

3. **`reduce_add`**: SSE3 平台使用两次 `_mm_hadd_ps` 实现水平求和；无 SSE3 时通过 shuffle + add 组合实现。Neon 使用原生 `vaddvq_f32`。

4. **`exp(float4)` 的已知 bug**: 实现中 `v.z` 和 `v.w` 都使用了 `expf(v.z)`（第 4 分量被忽略），这可能是一个 copy-paste 错误，但由于 w 分量在大多数使用场景中不重要，尚未修复。

## 关联文件

- `util/types_float4.h` — float4 结构体定义（包含 `__m128 m128` 成员）
- `util/math_float3.h` — float3 的 SSE 实现依赖 float4 的 shuffle
- `util/math_float8.h` — vfloat8 的 AVX 实现结构类似
