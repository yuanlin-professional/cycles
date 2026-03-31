# math_float3.h - 三维浮点向量运算

## 概述

`math_float3.h` 是 Cycles 中使用最广泛的向量数学头文件，为 `float3` 类型提供全面的数学运算支持。float3 在渲染器中表示位置、方向、法线、颜色(RGB)等核心数据。该文件在 CPU 上针对 SSE/SSE4.2/Neon 指令集提供了深度优化，同时保留标量回退路径以支持 GPU 后端。

## 核心函数

### 构造与初始化
| 函数 | 说明 |
|------|------|
| `zero_float3()` | 零向量（SSE 下使用 `_mm_setzero_ps`） |
| `one_float3()` | 单位向量 (1,1,1) |
| `reciprocal(a)` | 逐分量取倒数 |

### 运算符重载（均有 SSE 优化路径）
- 算术: `+`, `-`, `*`, `/`（float3 间及与标量）
- 复合赋值: `+=`, `-=`, `*=`, `/=`
- 比较: `==`, `!=`, `>=`, `<`, `>`
- 支持 `packed_float3` 的 `*=`, `/=`, `+=` 运算

### 向量数学
| 函数 | 说明 |
|------|------|
| `dot(a, b)` | 点积（SSE4.2 下使用 `_mm_dp_ps`） |
| `dot_xy(a, b)` | 仅计算 xy 分量的点积 |
| `cross(a, b)` | 叉积（SSE 下使用 shuffle + msub 优化） |
| `len(a)` / `len_squared(a)` | 长度 / 长度平方 |
| `normalize(a)` | 归一化（SSE4.2 下使用 `dp_ps + div_ps`） |
| `safe_normalize(a)` / `safe_normalize_fallback` | 安全归一化，带可选回退值 |
| `normalize_len(a, *t)` / `safe_normalize_len` | 归一化并输出原始长度 |
| `distance(a, b)` | 两点距离 |
| `project(v, v_proj)` | 向量投影 |
| `reflect` / `refract` / `faceforward` | 反射/折射/面朝向翻转 |

### 分量操作
| 函数 | 说明 |
|------|------|
| `reduce_min` / `reduce_max` / `reduce_add` | 分量规约 |
| `average(a)` | 分量均值 `(x+y+z)/3` |
| `min(a,b)` / `max(a,b)` / `clamp` | 逐分量 min/max/clamp |
| `fabs` / `fmod` / `sqrt` / `floor` / `ceil` / `round` | 逐分量数学函数 |
| `mix` / `interp` / `saturate` | 插值与饱和 |
| `exp` / `log` / `sin` / `cos` / `tan` / `atan2` | 逐分量超越函数 |
| `power` / `safe_pow` / `sqr` | 幂运算 |
| `safe_divide` / `safe_floored_fmod` / `safe_fmod` / `wrap` | 安全数学运算 |

### 几何工具
| 函数 | 说明 |
|------|------|
| `triangle_area(v1, v2, v3)` | 三角形面积 |
| `make_orthonormals(N, *a, *b)` | 从法线构造正交基 |
| `rotate_around_axis(p, axis, angle)` | 绕轴旋转（Rodrigues 公式） |
| `precise_angle(a, b)` | 高精度向量夹角（基于 Kahan 方法） |
| `tan_angle(a, b)` | 向量夹角的正切 |
| `map_to_tube` / `map_to_sphere` | 3D 坐标到 UV 的圆柱/球面投影 |

### 类型转换
| 函数 | 说明 |
|------|------|
| `float3_as_uint3` / `uint3_as_float3` | 位级别重解释 |
| `copy_v3_v3` | 复制到原始 float 数组 |
| `is_zero` / `any_zero` / `isfinite_safe` / `ensure_finite` | 状态检查 |
| `compatible_sign` / `isequal` / `isequal_mask` / `is_zero_mask` | 条件掩码生成 |
| `select(mask, a, b)` / `mask(mask, a)` | 条件选择（SSE blendv 优化） |

## 依赖关系

- **内部头文件**: `util/math_base.h`, `util/math_float4.h`, `util/types_float3.h`, `util/types_float4.h`, `util/types_int3.h`, `util/types_uint3.h`
- **被引用**: 通过 `util/math.h` 被几乎所有 Cycles 模块引用；直接被 `math_fast.h`, `math_intersect.h` 等引用

## 实现细节 / 关键算法

1. **SSE 优化**: float3 内部存储为 `__m128`（含第 4 分量 padding），大部分运算直接使用 128 位 SIMD 指令。叉积使用 `shuffle + msub` 组合实现，避免标量分支。

2. **`make_orthonormals`**: 给定法线 N，构造正交基 {N, a, b}。通过 `(1,1,1) x N` 获得第一个正交向量（当 N 的三个分量相同时退化为 `(-1,1,1) x N`），再通过叉积获得第二个。

3. **`precise_angle`**: 基于 Kahan 的 "Mangled Angles" 方法 `2*atan2(|a-b|, |a+b|)`，避免 `acos(dot(a,b))` 在小角度时的精度问题。

4. **Neon 支持**: ARM 平台通过 `__KERNEL_NEON__` 宏使用 Neon SIMD 指令（如 `vabsq_f32`, `vrndnq_f32`），在 Apple Silicon 上获得良好性能。

## 关联文件

- `util/types_float3.h` — float3 / packed_float3 结构体定义
- `util/math_float4.h` — float4 运算（float3 的 SSE 实现依赖 float4 的 shuffle）
- `util/math_base.h` — 标量基础运算
