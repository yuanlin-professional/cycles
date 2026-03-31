# math_fast.h - 快速近似数学函数

## 概述

`math_fast.h` 提供一组以精度换速度的数学函数近似实现，源自 OpenImageIO / SLEEF / Sony Pictures Imageworks 等开源项目。这些函数主要用于着色器计算中精度要求不苛刻但对性能敏感的场景。设计原则是尽量避免分支、仅使用单精度运算，以便直接移植到 SIMD/GPU 环境。

## 核心函数

### 三角函数
| 函数 | 说明 | 最大误差 |
|------|------|----------|
| `fast_sinf(x)` | 正弦近似 (SLEEF 参数规约 + 多项式) | ~1.19e-07 |
| `fast_cosf(x)` | 余弦近似 | ~4.33e-07 |
| `fast_sincosf(x, *sin, *cos)` | 同时计算正弦和余弦 | 同上 |
| `fast_tanf(x)` | 正切近似，有效范围 [-8192, 8192] | — |
| `fast_sinpif(x)` | `sin(x*pi)` 近似 | ~9.19e-04 |
| `fast_cospif(x)` | `cos(x*pi)` 近似 | ~9.19e-04 |

### 反三角函数
| 函数 | 说明 | 最大误差 |
|------|------|----------|
| `fast_acosf(x)` | 反余弦 | ~4.52e-05 |
| `fast_asinf(x)` | 反正弦 | ~4.51e-05 |
| `fast_atanf(x)` | 反正切 | ~6.56e-06 |
| `fast_atan2f(y, x)` | 二参数反正切 | ~6.56e-06 |

### 指数与对数
| 函数 | 说明 | 最大 ULP |
|------|------|----------|
| `fast_log2f(x)` | 以 2 为底的对数 | 3713596 |
| `fast_logf(x)` | 自然对数 | 5148137 |
| `fast_log10(x)` | 以 10 为底的对数 | 4471615 |
| `fast_exp2f(x)` | 2 的指数 | 232 |
| `fast_expf(x)` | 自然指数 | 230 |
| `fast_exp10(x)` | 10 的指数 | 232 |
| `fast_expm1f(x)` | `exp(x)-1`，小值优化 | — |
| `fast_exp2f4(x)` | float4 版 2 的指数（SSE4 SIMD） | — |

### 双曲函数
| 函数 | 说明 |
|------|------|
| `fast_sinhf` / `fast_coshf` / `fast_tanhf` | 双曲正弦/余弦/正切近似 |

### 特殊函数
| 函数 | 说明 |
|------|------|
| `fast_safe_powf(x, y)` | 安全快速幂（处理负底数和特殊值） |
| `fast_erff(x)` / `fast_erfcf(x)` | 误差函数及其补函数 |
| `fast_ierff(x)` | 反误差函数（Mike Giles 近似） |
| `fast_inv_cbrtf(x)` | 快速逆立方根（两次牛顿迭代） |

### 辅助函数
| 函数 | 说明 |
|------|------|
| `madd(a, b, c)` | `a*b+c`，可能被编译器优化为 FMA |
| `madd4(a, b, c)` | float4 版本 |
| `fast_rint(x)` | 快速四舍五入到最近整数 |
| `floor_log2f(x)` | 通过浮点位模式提取以 2 为底的对数 |
| `vector_angle(a, b)` | 基于 fast_atan2f 的向量夹角计算 |

## 依赖关系

- **内部头文件**: `util/math_base.h`, `util/math_float3.h`, `util/math_float4.h`, `util/math_int4.h`, `util/types_float3.h`, `util/types_float4.h`
- **被引用**: `util/types_rgbe.h`（RGBE 编解码）, kernel 中的 BSDF/光源采样（`bsdf_microfacet.h`, `bsdf_hair.h`, `light/spot.h` 等）, `scene/light_tree.cpp`

## 实现细节 / 关键算法

1. **SLEEF 参数规约**: `fast_sinf`/`fast_cosf` 使用 Cody-Waite 四段参数规约，将输入映射到 [-pi/2, pi/2]，再用 5-6 阶多项式逼近。在 [-2pi, 2pi] 范围内最大误差约 1 ULP。

2. **`fast_log2f`**: 将浮点数拆分为指数和尾数部分，对尾数使用 6 阶有理多项式逼近 `log2(1+f)`。97.46% 的值 ULP 差为 0。

3. **`fast_exp2f`**: 将输入拆分为整数和小数部分，小数部分用 5 阶多项式逼近 `2^x`，整数部分直接通过 IEEE 754 指数位移实现。

4. **`fast_safe_powf`**: 通过 `exp2(y * log2(|x|))` 实现，对负底数的整数幂进行特殊处理（检查指数位模式判断奇偶性）。

5. **`fast_inv_cbrtf`**: 使用类似 Quake III 快速逆平方根的技巧——通过浮点位操作获得初始估计值 `0x54a24242 - float_bits/3`，然后进行两次牛顿迭代 `y = (2/3)*y + 1/(3*y*y*x)`。

## 关联文件

- `util/math_base.h` — 基础标量运算和常量
- `kernel/closure/bsdf_microfacet.h` — 微表面 BSDF 中大量使用快速三角函数
- `kernel/light/` — 光源采样中使用快速指数和反三角函数
