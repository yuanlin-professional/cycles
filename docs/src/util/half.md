# half.h - 半精度浮点数（Half Float）转换工具

## 概述
本文件实现了 IEEE 754 半精度浮点数（16 位）与单精度浮点数（32 位）之间的转换工具，主要用于图像纹理的高效存储和显示纹理的快速转换。提供了标量和 SIMD 向量化（SSE/AVX2）两种转换路径，支持 CPU（含 Metal/OneAPI/CUDA/HIP）多种后端设备。

## 类与结构体

### `class half`
半精度浮点数类型（仅在非 CUDA/HIP/OneAPI/Metal 环境下定义）。实际存储为 `unsigned short`，通过类封装与普通 `unsigned short` 区分。
- 提供默认构造、从 `unsigned short` 的构造和赋值运算符
- CUDA/HIP 使用平台自有的 `half` 类型

### `struct half4`
四分量半精度浮点向量，成员为 `half x, y, z, w`。

## 核心函数

### 图像纹理转换
| 函数 | 说明 |
|------|------|
| `half float_to_half_image(float f)` | 单精度转半精度（图像存储用），消除 NaN/Inf，钳位到 65504 |
| `float half_to_float_image(half h)` | 半精度转单精度（图像采样用） |
| `float4 half4_to_float4_image(half4 h)` | 四分量半精度转单精度 |

### 显示纹理转换
| 函数 | 说明 |
|------|------|
| `half float_to_half_display(float f)` | 简化的单精度转半精度（假设非负、无 NaN/Inf，denormal 视为零） |
| `half4 float4_to_half4_display(float4 f)` | 四分量版本，含 SSE/AVX2 优化路径 |

### 平台特定实现
- **Metal**: 使用 `half_to_float()` 自定义实现
- **CUDA/HIP**: 使用内置 `__float2half()` / `__half2float()`
- **OneAPI**: 使用隐式类型转换
- **CPU (SSE)**: 位操作实现，`float4_to_half4_display` 使用 `_mm_packs_epi32`
- **CPU (AVX2)**: `float4_to_half4_display` 使用 `_mm_cvtps_ph` 硬件指令

## 依赖关系
- **内部头文件**: `util/math.h`, `util/types.h`, `util/simd.h`（SSE2 时）
- **被引用**: `util/image.h`, `kernel/` 下的纹理采样代码, 显示缓冲区管理模块

## 关联文件
- `util/image.h` - 使用 `half` 作为图像像素类型之一
- `util/color.h` - 颜色转换（常与半精度转换配合使用）
- `util/simd.h` - SIMD 指令集封装
