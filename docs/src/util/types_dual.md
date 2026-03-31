# types_dual.h - 对偶数（Dual Number）类型定义，用于自动微分

## 概述

`types_dual.h` 定义了用于自动微分的对偶数模板结构体 `dual<T>`，每个对偶数包含一个值 `val` 及其在屏幕空间 x/y 方向上的偏导数 `dx` 和 `dy`。该文件为 `float`、`float2`、`float3`、`float4` 提供了模板特化版本，并定义了便捷的类型别名 `dual1`、`dual2`、`dual3`、`dual4`。对偶数类型在着色器求导（纹理过滤、凹凸映射等）中广泛使用。

## 类与结构体

### `template<class T> struct dual`

通用对偶数模板，包含三个成员：

| 成员 | 类型 | 说明 |
|------|------|------|
| `val` | `T` | 实际值 |
| `dx` | `T` | 对屏幕空间 x 方向的偏导数 |
| `dy` | `T` | 对屏幕空间 y 方向的偏导数 |

**构造函数**：
- 默认构造（零初始化）
- `explicit dual(T val)` — 仅设置值，导数为零
- `dual(T val, T dx, T dy)` — 完整构造

### 模板特化

针对 `float2`、`float3`、`float4` 提供了显式特化，使用对应的 `make_float*` 函数进行零初始化。

### 类型别名

| 别名 | 展开 |
|------|------|
| `dual1` | `dual<float>` |
| `dual2` | `dual<float2>` |
| `dual3` | `dual<float3>` |
| `dual4` | `dual<float4>` |

## 核心函数/运算符重载

| 函数签名 | 功能说明 |
|----------|----------|
| `dual3 make_float3(const dual<T> &a)` | 将任意对偶数类型提升为 `dual3` |
| `dual3 make_float3(dual1 a, dual1 b, dual1 c)` | 从三个标量对偶数组装 `dual3` |
| `dual4 make_float4(dual3 a)` | 将 `dual3` 扩展为 `dual4`（w 分量导数为 0） |
| `dual4 make_homogeneous(dual3 a)` | 将 `dual3` 转为齐次坐标 `dual4`（val.w=1, dx.w=dy.w=0） |
| `void print_dual1(const char*, dual1)` | 调试打印标量对偶数 |
| `void print_dual2(const char*, dual2)` | 调试打印二维对偶数 |
| `void print_dual3(const char*, dual3)` | 调试打印三维对偶数 |

## 依赖关系

- **内部头文件**:
  - `util/types_float2.h`
  - `util/types_float3.h`
  - `util/types_float4.h`
- **被引用**: `util/types.h`、`util/math_dual.h`

## 关联文件

- `src/util/math_dual.h` — 对偶数的算术运算（加减乘除、链式法则等）
- `src/util/types.h` — 类型聚合头文件
- `src/kernel/svm/` — 着色器虚拟机中大量使用对偶数进行纹理微分
