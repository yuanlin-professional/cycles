# math_float2.h - 二维浮点向量运算

## 概述

`math_float2.h` 为 `float2` 类型提供完整的数学运算支持，包括算术运算符重载、向量数学函数（点积、叉积、长度、归一化等）以及分量级的数学操作。float2 在 Cycles 中主要用于 UV 纹理坐标、2D 屏幕空间计算以及极坐标转换。

## 核心函数

### 构造与初始化
| 函数 | 说明 |
|------|------|
| `zero_float2()` | 返回 (0, 0) |
| `one_float2()` | 返回 (1, 1) |
| `make_zero<float2>()` | 模板特化零值 |

### 运算符重载
- 算术: `+`, `-`, `*`, `/`（float2 与 float2 或 float 标量）
- 复合赋值: `+=`, `*=`, `/=`
- 比较: `==`, `!=`, `>=`

### 向量数学
| 函数 | 说明 |
|------|------|
| `dot(a, b)` | 点积 |
| `cross(a, b)` | 2D 叉积（返回标量，即行列式） |
| `len(a)` / `len_squared(a)` | 向量长度 / 长度的平方 |
| `normalize(a)` / `safe_normalize(a)` | 归一化 / 安全归一化（避免零向量除零） |
| `normalize_len(a, *t)` | 归一化并输出原始长度 |
| `distance(a, b)` | 两点欧几里得距离 |

### 分量操作
| 函数 | 说明 |
|------|------|
| `reduce_min` / `reduce_max` / `reduce_add` | 分量最小值/最大值/求和 |
| `average(a)` | 分量均值 `(x+y)/2` |
| `min(a, b)` / `max(a, b)` / `clamp` | 逐分量最小/最大/截断 |
| `fabs(a)` / `fmod(a, b)` / `floor(a)` | 逐分量绝对值/取模/向下取整 |
| `interp(a, b, t)` / `mix(a, b, t)` | 逐分量线性插值 |
| `power(v, e)` | 逐分量幂运算 |
| `select(mask, a, b)` / `mask(mask, a)` | 条件选择 |

### 工具函数
| 函数 | 说明 |
|------|------|
| `is_zero(a)` | 判断是否为零向量 |
| `isequal(a, b)` | Metal 兼容的相等判断 |
| `as_float2(float4)` | 从 float4 提取前两个分量 |
| `safe_divide_float2_float` | 安全除法（避免除零） |

## 依赖关系

- **内部头文件**: `util/math_base.h`, `util/types_float2.h`, `util/types_float4.h`
- **被引用**: 通过 `util/math.h` 被间接引用；在纹理坐标映射、降噪像素坐标等场景广泛使用

## 实现细节 / 关键算法

- 所有运算均为纯标量实现（无 SIMD），因为 float2 的 SIMD 化收益有限
- Metal 平台下部分运算符（如 `==`, `dot`）由 Metal Shading Language 内建提供，通过 `__KERNEL_METAL__` 条件编译跳过
- `power` 函数有意不命名为 `pow`，因为 HIP 编译器在名称修饰(name mangling)时会崩溃

## 关联文件

- `util/types_float2.h` — float2 结构体定义
- `util/math_float3.h` — float3 运算（float2 经常在 3D 运算中用于存储中间 2D 结果）
- `util/math_base.h` — 标量 min/max/clamp 等基础函数
