# int_vector_types.h - 整数向量类型定义头文件

## 概述

该头文件为 Cycles 渲染器的开放着色语言（OSL）着色器系统定义了整数向量类型（`int2`、`int3`、`int4`），并重载了这些类型的基本算术和位运算操作符。同时提供了整数向量与浮点向量之间的相互转换函数。这些类型主要用于需要整数坐标运算的场景，如哈希计算和噪声函数。

## 宏定义/函数/类型

### 结构体类型

#### `int2`
二维整数向量，包含成员 `x`、`y`。

#### `int3`
三维整数向量，包含成员 `x`、`y`、`z`。

#### `int4`
四维整数向量，包含成员 `x`、`y`、`z`、`w`。

### 操作符重载

每种整数向量类型均支持以下操作符（以 `int2` 为例，`int3` 和 `int4` 类似）：

| 操作符函数 | 说明 | 变体 |
|------------|------|------|
| `__operator__add__` | 加法 | `int2 + int2`，`int2 + int` |
| `__operator__mul__` | 乘法 | `int2 * int2`，`int2 * int` |
| `__operator__shr__` | 右移位 | `int2 >> int` |
| `__operator__xor__` | 异或 | `int2 ^ int2` |
| `__operator__bitand__` | 按位与 | `int2 & int` |

### 类型转换函数

| 函数名 | 说明 |
|--------|------|
| `vec2_to_int2(vector2 k)` | 将 `vector2` 转换为 `int2`（截断取整） |
| `int2_to_vec2(int2 k)` | 将 `int2` 转换为 `vector2` |
| `vec3_to_int3(point k)` | 将 `point` 转换为 `int3`（截断取整） |
| `int3_to_vec3(int3 k)` | 将 `int3` 转换为 `point` |
| `vec4_to_int4(vector4 k)` | 将 `vector4` 转换为 `int4`（截断取整） |
| `int4_to_vec4(int4 k)` | 将 `int4` 转换为 `vector4` |

## 依赖关系

- **上游依赖**：
  - `vector2.h` - 提供 `vector2` 类型定义
  - `vector4.h` - 提供 `vector4` 类型定义
- **被依赖**：主要被哈希（`node_hash.h`）和噪声（`node_noise.h`）相关着色器引用
- **许可证**：Apache-2.0，Blender Foundation 2025
