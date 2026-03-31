# color.h - 颜色空间转换与颜色运算工具

## 概述
本文件提供 Cycles 渲染器中颜色相关的核心工具函数，包括浮点数与字节之间的颜色值转换、sRGB 与线性色彩空间的互相转换、RGB/HSV/HSL 之间的色彩空间转换、CIE xyY 到 XYZ 的转换，以及安全除法等颜色运算。所有函数均标记为 `ccl_device` 或 `ccl_device_inline`，支持在 CPU 和 GPU（CUDA/HIP/Metal/OneAPI）上运行。

## 类与结构体
本文件无类或结构体定义，全部为设备端内联函数。

## 核心函数

### 浮点 / 字节转换
| 函数 | 说明 |
|------|------|
| `uchar float_to_byte(float val)` | 浮点 [0,1] 转 uchar [0,255]，含边界钳位 |
| `float byte_to_float(uchar val)` | uchar 转浮点 |
| `uchar4 color_float_to_byte(float3 c)` | float3 颜色转 uchar4（alpha=0） |
| `uchar4 color_float4_to_uchar4(float4 c)` | float4 颜色转 uchar4 |
| `float3 color_byte_to_float(uchar4 c)` | uchar4 转 float3 |
| `float4 color_uchar4_to_float4(uchar4 c)` | uchar4 转 float4 |

### sRGB / 线性空间转换
| 函数 | 说明 |
|------|------|
| `float color_srgb_to_linear(float c)` | 单通道 sRGB 转线性（分段函数，阈值 0.04045） |
| `float color_linear_to_srgb(float c)` | 单通道线性转 sRGB（分段函数，阈值 0.0031308） |
| `float3 color_srgb_to_linear_v3(float3 c)` | float3 版本 |
| `float3 color_linear_to_srgb_v3(float3 c)` | float3 版本 |
| `float4 color_srgb_to_linear_v4(float4 c)` | float4 版本（SSE2 优化路径可用时自动启用） |
| `float4 color_linear_to_srgb_v4(float4 c)` | float4 版本 |
| `float4 color_srgb_to_linear_sse2(float4 &c)` | SSE2 优化的批量转换，使用快速 `powf(x, 2.4)` 近似 |

### 色彩空间转换
| 函数 | 说明 |
|------|------|
| `float3 rgb_to_hsv(float3 rgb)` | RGB 转 HSV |
| `float3 hsv_to_rgb(float3 hsv)` | HSV 转 RGB |
| `float3 rgb_to_hsl(float3 rgb)` | RGB 转 HSL |
| `float3 hsl_to_rgb(float3 hsl)` | HSL 转 RGB |
| `float3 xyY_to_xyz(float x, float y, float Y)` | CIE xyY 转 XYZ 色彩空间 |

### 颜色运算
| 函数 | 说明 |
|------|------|
| `float3 color_highlight_compress(float3 color, float3 *variance)` | 高光压缩（对数变换），同时更新方差 |
| `float3 color_highlight_uncompress(float3 color)` | 高光解压缩（指数变换） |
| `Spectrum safe_invert_color(Spectrum a)` | 安全的颜色倒数（零通道返回 0） |
| `Spectrum safe_divide_color(Spectrum a, Spectrum b, float fallback)` | 安全的颜色除法 |
| `float3 safe_divide_even_color(float3 a, float3 b)` | 安全除法，零通道使用其余通道的均值保持灰度一致性 |

### SSE2 快速幂函数（`__KERNEL_SSE2__`）
| 函数 | 说明 |
|------|------|
| `float4 fastpow_sse2<exp, e2coeff>(float4)` | 基于浮点表示的快速幂初始猜测 |
| `float4 improve_5throot_solution_sse2(float4, float4)` | 牛顿-拉弗森法改进 5 次根近似 |
| `float4 fastpow24_sse2(float4)` | 快速计算 `pow(x, 2.4)`，精度优于 GLIBC 的 `powf` |

## 依赖关系
- **内部头文件**: `util/math.h`, `util/types.h`
- **被引用**: `kernel/` 下的着色器和光照模块, `scene/shader.cpp`, `render/` 下的后处理模块

## 关联文件
- `util/math.h` - 数学基础函数
- `util/types.h` - `float3`, `float4`, `uchar4`, `Spectrum` 等类型定义
- `util/half.h` - 半精度浮点转换
