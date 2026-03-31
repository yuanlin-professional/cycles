# math_base.h - 标量数学基础运算

## 概述

`math_base.h` 提供 Cycles 渲染器中最底层的标量（`float`、`int`、`uint`）数学函数，包括常量定义、基本算术运算、类型转换、插值、安全数学函数以及位操作。该文件同时支持 CPU 和 GPU（CUDA/HIP/Metal/oneAPI）编译路径，通过条件宏选择最优的平台实现。

## 核心函数

### 常量定义
- `M_PI_F`, `M_2PI_F`, `M_PI_2_F`, `M_1_PI_F` 等 — 单精度浮点 Pi 系列常量
- `M_SQRT2_F`, `M_SQRT3_F`, `M_LN2_F`, `M_LN10_F` — 常用数学常量

### 标量基本运算
| 函数 | 说明 |
|------|------|
| `min` / `max` | 支持 int, uint32, uint64, float, double 及 size_t 的模板重载 |
| `min4` / `max4` | 四个值取最小/最大 |
| `clamp(a, mn, mx)` | 区间截断 |
| `mix(a, b, t)` / `interp` | 线性插值 |
| `smoothstep` / `smoothstepf` | 三次 Hermite 平滑阶跃 |
| `saturatef(a)` | 截断到 [0, 1] |

### 类型转换与位操作
| 函数 | 说明 |
|------|------|
| `as_int` / `as_uint` / `as_float` | 位级别类型重解释（union-based） |
| `__float_as_int` / `__int_as_float` | GPU 风格位转换 |
| `popcount` | 位计数（支持 GCC 内建、MSVC 手动、GPU 各平台） |
| `count_leading_zeros` / `count_trailing_zeros` | 前导/尾随零计数 |
| `find_first_set` | 查找最低位 1 |
| `reverse_integer_bits` | 32 位整数位反转 |
| `next_power_of_two` / `prev_power_of_two` | 2 的幂次对齐 |

### 安全数学函数（NaN-safe）
| 函数 | 说明 |
|------|------|
| `isnan_safe` / `isfinite_safe` | 兼容快速数学模式的安全检测 |
| `ensure_finite` | 非有限值替换为 0 |
| `safe_sqrtf` / `safe_acosf` / `safe_asinf` | 输入截断到有效域 |
| `safe_powf` / `safe_divide` / `safe_logf` / `safe_modulo` | 防除零、防 NaN |
| `compatible_powf` / `compatible_atan2` / `compatible_signf` | 跨平台行为一致性封装 |

### 插值与数值工具
| 函数 | 说明 |
|------|------|
| `inverse_lerp(a, b, x)` | 反向线性插值 |
| `cubic_interp` | 三次 Catmull-Rom 插值 |
| `wrapf` / `pingpongf` | 循环映射与来回弹跳 |
| `smoothminf` | 平滑最小值（基于三次核） |
| `sqr` / `sin_from_cos` / `cos_from_sin` | 常用三角恒等式快捷运算 |
| `beta(x, y)` | Beta 函数 |
| `solve_quadratic` | 求解一元二次方程（数值稳定版） |
| `compare_floats` | 基于 ULP 和绝对差的浮点比较 |

### 数据结构
- **`Interval<T>`** — 闭区间 [min, max]，支持判空、包含检测、交集运算
- **`Extrema<T>`** — 最小/最大值对，支持合并、加法、标量乘法

## 依赖关系

- **内部头文件**: `util/defines.h`, `util/types_base.h`
- **系统头文件**: `<cfloat>`, `<cmath>`（非 Metal 平台）
- **被引用**: 被所有 `math_float*.h`, `math_int*.h`, `math_dual.h`, `math_fast.h`, `math_cdf.h`, `rect.h` 引用；间接被整个 kernel 和 scene 模块使用

## 实现细节 / 关键算法

1. **`isnan_safe` / `isfinite_safe`**: 在启用 `-ffast-math` 的编译环境中，标准库 `isnan()`/`isfinite()` 可能被优化掉。这里通过位模式检测实现，确保在任何优化级别下都能正确检测。

2. **`solve_quadratic`**: 采用数值稳定的求根公式——先计算 `temp = -0.5*(b + copysign(sqrt(disc), b))`，避免了 `b` 与 `sqrt(disc)` 接近时的灾难性抵消，再通过韦达定理 `x1*x2 = c/a` 求另一根。

3. **`reverse_integer_bits`**: 多平台优化，优先使用 CUDA `__brev`、Metal `reverse_bits`、ARM `rbit` 指令，CPU 上回退到 pairwise 交换 + `bswap` 方案。

4. **`sin_sqr_to_one_minus_cos`** / **`one_minus_cos`**: 在小角度场景使用二阶泰勒展开 `1-cos(x) ≈ x^2/2`，避免大数减小数导致的精度损失。

## 关联文件

- `util/types_base.h` — 基础类型 (float, int, uint) 定义
- `util/defines.h` — 编译器/平台宏定义（CCL_NAMESPACE, ccl_device_inline 等）
- `util/math.h` — 聚合导出本文件
