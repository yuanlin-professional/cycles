# math_dual.h - 对偶数（自动微分）运算

## 概述

`math_dual.h` 为 Cycles 中的对偶数类型 `dual<T>` 提供数学运算支持。对偶数携带一个值 (`val`) 及其对屏幕空间 x/y 方向的偏导数 (`dx`, `dy`)，用于纹理过滤时自动计算 UV 微分。该文件定义了对偶数的标量乘法、取负、求平均、规约求和及点积运算。

## 核心函数

| 函数 | 说明 |
|------|------|
| `make_zero<dual1/2/3/4>()` | 各维度对偶数的零值特化 |
| `operator*(dual<T1> a, T2 b)` | 对偶数与标量的乘法，同时缩放 val、dx、dy |
| `operator-(dual<T> a)` | 对偶数取负 |
| `average(dual<T> a)` | 对 val、dx、dy 分别求分量均值，返回 dual1 |
| `reduce_add(dual<T> a)` | 对 val、dx、dy 分别求分量和，返回 dual1 |
| `dot(dual<T1> a, T2 b)` | 对偶数与普通向量的点积，等价于 `reduce_add(a * b)` |

## 依赖关系

- **内部头文件**: `util/math_base.h`, `util/types_dual.h`
- **被引用**: 通过 `util/math.h` 被间接引用；直接用于 kernel 中的纹理采样和法线贴图微分计算

## 实现细节 / 关键算法

对偶数算术遵循自动微分的前向模式规则：
- 乘法: `(a, a') * b = (a*b, a'*b)` — 常量乘法不改变导数的形式，仅线性缩放
- 取负: `-(a, a') = (-a, -a')`
- 点积: 通过先乘后规约实现，保持导数链式法则的正确性

`dual1/dual2/dual3/dual4` 分别对应 `dual<float>`, `dual<float2>`, `dual<float3>`, `dual<float4>`，在纹理坐标和颜色微分中广泛使用。

## 关联文件

- `util/types_dual.h` — `dual<T>` 结构体定义
- `util/math_float3.h` — float3 运算（dual3 的 val/dx/dy 类型）
- `kernel/svm/tex_coord.h` — 使用对偶数计算纹理坐标微分的典型场景
