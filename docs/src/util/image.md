# image.h - 图像像素操作工具与 OIIO 接口封装

## 概述
本文件封装了 OpenImageIO（OIIO）的图像 I/O 接口，并提供了一组模板化的图像像素操作工具函数，支持在不同像素类型（`float`、`uchar`、`uint16_t`、`half`）之间进行类型转换、原生格式乘法运算和图像缩放。是 Cycles 图像纹理加载管线的基础模块。

## 类与结构体
无自定义类。通过 `using` 引入 OIIO 类型：

| 类型 | 说明 |
|------|------|
| `ImageInput` | OIIO 图像输入接口 |
| `ImageOutput` | OIIO 图像输出接口 |
| `ImageSpec` | 图像规格描述（分辨率、通道数、数据类型等） |
| `AutoStride` | 自动步长计算 |

## 核心函数

### 像素类型转换
| 函数 | 说明 |
|------|------|
| `float util_image_cast_to_float<T>(T value)` | 将任意像素类型转为 float（模板特化） |
| `T util_image_cast_from_float<T>(float value)` | 将 float 转为指定像素类型（含钳位） |

特化版本覆盖：`float`（直传）、`uchar`（/255）、`uint16_t`（/65535）、`half`（通过 `half_to_float_image`）

### 原生像素乘法
| 函数 | 说明 |
|------|------|
| `T util_image_multiply_native<T>(T a, T b)` | 在原生数据格式中进行乘法运算 |

- `float`: 直接乘法
- `uchar`: `(a * b) / 255`
- `uint16_t`: `(a * b) / 65535`
- `half`: 先转 float 相乘再转回 half

### 图像缩放
| 函数 | 说明 |
|------|------|
| `void util_image_resize_pixels<T>(...)` | 按比例缩放图像像素数据，支持缩小（box 滤波），放大尚未实现 |

## 依赖关系
- **内部头文件**: `util/half.h`, `util/vector.h`
- **外部依赖**: `<OpenImageIO/imageio.h>`
- **被引用**: `scene/image.cpp`, `scene/image_oiio.cpp`

## 关联文件
- `util/image_impl.h` - 图像缩放的模板实现（自动包含）
- `util/half.h` - 半精度浮点数支持
- `util/vector.h` - 像素数据容器
