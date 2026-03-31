# node_hash.h - 哈希函数库头文件

## 概述

该头文件是开放着色语言(OSL)着色器系统的哈希函数库。提供将整数向量或浮点数/向量哈希映射到 [0, 1] 范围内浮点数、向量或颜色的全套函数。整数版本基于 PCG（Permuted Congruential Generator）哈希算法实现，浮点版本基于 OSL 内置的 `hashnoise` 函数。被 Voronoi 纹理、白噪声纹理、Gabor 纹理等多个着色器节点引用。

## 文件依赖

```
node_hash.h
  ├── int_vector_types.h   (int2/int3/int4 整数向量类型)
  ├── stdcycles.h           (Cycles 标准库)
  ├── vector2.h             (vector2 类型)
  └── vector4.h             (vector4 类型)
```

## 整数哈希函数（PCG 算法）

基于 PCG 2D/3D/4D 哈希的有符号整数实现，核心步骤为：乘以 1664525，加 1013904223，交叉混合各分量，右移 16 位异或扰动。

| 函数签名 | 说明 |
|----------|------|
| `vector2 hash_int2_to_vector2(int2 k)` | 2D 整数向量 -> 2D 浮点向量 [0,1] |
| `vector3 hash_int3_to_vector3(int3 k)` | 3D 整数向量 -> 3D 浮点向量 [0,1] |
| `vector4 hash_int4_to_vector4(int4 k)` | 4D 整数向量 -> 4D 浮点向量 [0,1] |
| `color hash_int2_to_color(int2 k)` | 2D 整数向量 -> 颜色 [0,1] |
| `color hash_int4_to_color(int4 k)` | 4D 整数向量 -> 颜色 [0,1] |

## 浮点哈希到标量函数

使用 OSL 内置 `hashnoise` 函数实现。

| 函数签名 | 说明 |
|----------|------|
| `float hash_float_to_float(float k)` | 标量 -> 标量 [0,1] |
| `float hash_vector2_to_float(vector2 k)` | 2D 向量 -> 标量 [0,1] |
| `float hash_vector3_to_float(vector3 k)` | 3D 向量 -> 标量 [0,1] |
| `float hash_vector4_to_float(vector4 k)` | 4D 向量 -> 标量 [0,1] |

## 浮点哈希到向量函数

通过对输入附加不同常数（如 1.0、2.0）或重新排列分量进行多次哈希，生成多维输出。

| 函数签名 | 说明 |
|----------|------|
| `vector2 hash_float_to_vector2(float k)` | 标量 -> 2D 向量 [0,1] |
| `vector2 hash_vector2_to_vector2(vector2 k)` | 2D -> 2D 向量 [0,1] |
| `vector2 hash_vector3_to_vector2(vector3 k)` | 3D -> 2D 向量 [0,1] |
| `vector2 hash_vector4_to_vector2(vector4 k)` | 4D -> 2D 向量 [0,1] |
| `vector3 hash_float_to_vector3(float k)` | 标量 -> 3D 向量 [0,1] |
| `vector3 hash_vector2_to_vector3(vector2 k)` | 2D -> 3D 向量 [0,1] |
| `vector3 hash_vector4_to_vector3(vector4 k)` | 4D -> 3D 向量 [0,1] |
| `vector4 hash_vector4_to_vector4(vector4 k)` | 4D -> 4D 向量 [0,1] |

## 浮点哈希到颜色函数

| 函数签名 | 说明 |
|----------|------|
| `color hash_float_to_color(float k)` | 标量 -> 随机颜色 [0,1] |
| `color hash_vector2_to_color(vector2 k)` | 2D 向量 -> 随机颜色 [0,1] |
| `color hash_vector3_to_color(vector3 k)` | 3D 向量 -> 随机颜色 [0,1] |
| `color hash_vector4_to_color(vector4 k)` | 4D 向量 -> 随机颜色 [0,1] |

## 实现特点

- **PCG 整数哈希**：使用乘法常数 1664525 和加法常数 1013904223（即 LCG 参数），通过分量交叉乘法和位异或实现快速高质量整数哈希。最终通过 `& 0x7FFFFFFF` 取正整数并归一化到 [0, 1]。
- **浮点哈希**：基于 OSL 内置 `hashnoise` 函数，通过添加常数偏移或重排分量来生成多个独立的哈希通道。
- **分量重排**：4D 到颜色/向量的转换通过置换 `(x,y,z,w)` -> `(z,x,w,y)` -> `(w,z,y,x)` 等方式生成三个独立哈希值。

## 被引用文件

- `node_voronoi.h` - Voronoi 基础函数库
- `node_white_noise_texture.osl` - 白噪声纹理着色器
- `node_gabor_texture.osl` - Gabor 噪声纹理着色器
