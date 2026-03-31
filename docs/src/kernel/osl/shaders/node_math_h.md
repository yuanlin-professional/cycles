# node_math.h - 数学运算辅助函数头文件

## 概述

该头文件为 Cycles 渲染器的开放着色语言（OSL）着色器提供通用的数学运算辅助函数。包含安全除法、安全取模、平方根、对数等防止除零和无效输入的安全函数，以及向量投影、对齐、循环映射、欧拉角转旋转矩阵等几何运算函数。被 `node_math.osl`、`node_vector_math.osl`、`node_vector_rotate.osl`、`node_hsv.osl` 等多个着色器引用。

## 包含的依赖

```osl
#include "vector2.h"
#include "vector4.h"

#define vector3 point
```

引入 `vector2` 和 `vector4` 扩展类型支持，并将 `vector3` 定义为 `point` 类型的别名。

## 函数列表

### 安全算术函数

| 函数签名 | 说明 |
|----------|------|
| `float safe_divide(float a, float b)` | 安全除法。`b == 0` 时返回 0.0，否则返回 `a / b` |
| `vector safe_divide(vector a, vector b)` | 逐分量安全除法。每个分量独立判断除数是否为零 |
| `float safe_modulo(float a, float b)` | 安全取模。`b == 0` 时返回 0.0，否则返回 `fmod(a, b)` |
| `float safe_floored_modulo(float a, float b)` | 向下取整安全取模。`b == 0` 时返回 0.0，否则返回 `a - floor(a/b) * b` |
| `float safe_sqrt(float a)` | 安全平方根。`a <= 0` 时返回 0.0 |
| `float safe_log(float a, float b)` | 安全对数。`a <= 0` 或 `b <= 0` 时返回 0.0，否则返回 `log(a) / log(b)` |

### 基本数学函数

| 函数签名 | 说明 |
|----------|------|
| `float sqr(float a)` | 平方函数，返回 `a * a` |
| `float fract(float a)` | 小数部分，返回 `a - floor(a)` |
| `float pingpong(float a, float b)` | 乒乓映射，值在 0 和 `b` 之间来回弹跳 |
| `float smoothmin(float a, float b, float c)` | 平滑最小值函数（基于 Inigo Quilez 的算法），`c` 控制平滑半径 |

### 取整函数

| 函数签名 | 说明 |
|----------|------|
| `vector2 round(vector2 a)` | vector2 类型的四舍五入 |
| `vector4 round(vector4 a)` | vector4 类型的四舍五入 |

注意：`float` 和 `vector3` 类型的 `round()` 已在 `stdosl.h` 中定义。

### 向量几何函数

| 函数签名 | 说明 |
|----------|------|
| `vector project(vector v, vector v_proj)` | 向量投影。将 `v` 投影到 `v_proj` 方向上。`v_proj` 长度为零时返回零向量 |
| `vector snap(vector a, vector b)` | 向量对齐。将 `a` 的各分量对齐到 `b` 对应分量的整数倍 |
| `float wrap(float value, float max, float min)` | 标量循环映射（改编自 GODOT 引擎） |
| `point wrap(point value, point max, point min)` | 逐分量循环映射 |
| `point compatible_faceforward(point vec, point incident, point reference)` | 兼容版面朝向函数，使用 `dot(reference, incident) < 0` 的 GLSL 风格判断 |

### 矩阵与变换函数

| 函数签名 | 说明 |
|----------|------|
| `matrix euler_to_mat(point euler)` | 将欧拉角（XYZ 顺序）转换为 3x3 旋转矩阵（存储在 4x4 矩阵中） |

### 辅助函数

| 函数签名 | 说明 |
|----------|------|
| `float average(point a)` | 计算三分量的算术平均值 |

## 关键实现细节

### smoothmin 函数
基于 Inigo Quilez 的平滑最小值算法（参见 https://www.iquilezles.org/www/articles/smin/smin.htm ）。当平滑参数 `c != 0` 时，使用三次多项式在两个值之间平滑过渡：
```
h = max(c - abs(a - b), 0) / c
result = min(a, b) - h^3 * c / 6
```

### compatible_faceforward 函数
与开放着色语言（OSL）内置的 `faceforward` 行为不同：OSL 内置版本使用 `dot(I, Nref) > 0`，而此函数使用 GLSL 风格的 `dot(Nref, I) < 0`，在零值边界处的行为有差异。

### euler_to_mat 函数
采用 XYZ 内旋顺序（先绕 X 轴，再绕 Y 轴，最后绕 Z 轴），生成的旋转矩阵按行存储。
