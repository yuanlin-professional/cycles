# math_float8.h - 八维浮点向量运算（AVX）

## 概述

`math_float8.h` 为 `vfloat8` 类型提供数学运算支持。vfloat8 是 8 路宽度的浮点 SIMD 向量，在 AVX 平台上映射到 `__m256` 寄存器。该类型主要用于 Cycles 中的光谱渲染路径（spectral rendering），在该模式下颜色通过 8 个波长采样点表示。

## 核心函数

### 构造与初始化
| 函数 | 说明 |
|------|------|
| `zero_vfloat8()` | 零向量（AVX: `_mm256_setzero_ps`） |
| `one_vfloat8()` | 全 1 向量 |

### 运算符重载（AVX 优化）
- 算术: `+`, `-`, `*`, `/`, `^` (XOR)
- 复合赋值: `+=`, `-=`, `*=`, `/=`
- 一元取负
- 比较: `==`

### 向量数学
| 函数 | 说明 |
|------|------|
| `dot(a, b)` | 8 路点积（AVX: `_mm256_dp_ps` + 高低 lane 求和） |
| `sqrt(a)` / `sqr(a)` | 逐分量平方根 / 平方 |
| `pow(v, e)` | 逐分量幂运算 |

### 分量操作
| 函数 | 说明 |
|------|------|
| `reduce_add(a)` | 分量求和（AVX: 两次 `_mm256_hadd_ps`） |
| `reduce_min(a)` / `reduce_max(a)` | 分量极值 |
| `average(a)` | 分量均值 `sum/8` |
| `min(a,b)` / `max(a,b)` / `clamp` | 逐分量 min/max/clamp |
| `fabs(a)` | 逐分量绝对值 |
| `mix` / `saturate` / `exp` / `log` | 插值、饱和、超越函数 |
| `safe_divide` | 安全除法 |
| `ensure_finite` / `isfinite_safe` | 有限值检查 |
| `is_zero(a)` / `isequal(a, b)` | 状态检查 |
| `select(mask, a, b)` | 条件选择（AVX: `_mm256_blendv_ps`） |

### 类型转换与 SIMD 工具（`__KERNEL_SSE__`）
| 函数 | 说明 |
|------|------|
| `cast(vfloat8)` → `vint8` | 位级别重解释 |
| `low(a)` / `high(a)` | 提取低/高 128 位为 float4 |
| `shuffle<...>(a)` | 多种模板化重排（`_mm256_permutevar_ps`, `_mm256_shuffle_ps`） |
| `extract<i>(a)` | 提取第 i 个分量 |

## 依赖关系

- **内部头文件**: `util/math_base.h`, `util/types_float8.h`, `util/types_int8.h`
- **被引用**: 通过 `util/math.h` 被间接引用；在光谱渲染路径中被 kernel 着色节点使用

## 实现细节 / 关键算法

1. **AVX 256 位操作**: 所有基本算术运算在 AVX 下使用 `_mm256_*_ps` 单指令完成 8 路并行运算。非 AVX 平台回退到逐分量标量运算。

2. **`dot` 的实现**: AVX 的 `_mm256_dp_ps` 在 256 位向量上工作，但其结果存储在两个 128 位 lane 中，因此需要额外取 `[0]` 和 `[4]` 并相加以获得完整的 8 路点积。

3. **`low` / `high` 提取**: 使用 `_mm256_extractf128_ps` 将 256 位向量拆分为两个 128 位 float4。非 AVX 回退路径的顺序与 AVX 版本相反（这是因为 vfloat8 的内存布局在不同平台上的成员命名差异）。

4. **超越函数**: `exp` 和 `log` 没有 AVX 向量化版本，回退到 8 次标量调用。这是因为 AVX 缺少原生的指数/对数指令。

## 关联文件

- `util/types_float8.h` — vfloat8 结构体定义（含 `__m256 m256` 成员）
- `util/math_int8.h` — 配套的 vint8 整数运算
- `util/math_float4.h` — 结构上类似的 128 位版本
