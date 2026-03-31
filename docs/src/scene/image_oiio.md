# image_oiio.h / image_oiio.cpp - 基于 OpenImageIO 的图像文件加载器

## 概述

本文件实现了基于 OpenImageIO (OIIO) 库的图像加载器 `OIIOImageLoader`，是 Cycles 中最常用的图像加载方式。它负责从磁盘文件读取各种格式的图像（PNG、JPEG、TIFF、EXR、DPX 等），处理元数据检测、像素读取、CMYK 转换和 alpha 预乘关联。

## 类与结构体

### OIIOImageLoader
- **继承**: `ImageLoader`
- **功能**: 通过 OpenImageIO 库从文件系统加载图像数据
- **关键成员**:
  - `filepath` — 图像文件路径（`ustring` 类型）
- **关键方法**:
  - `load_metadata()` — 打开图像文件读取 ImageSpec，检测宽高、通道数、数据类型（float/half/byte/ushort）和色彩空间提示信息，不加载像素
  - `load_pixels()` — 通过 OIIO 读取像素数据，处理 CMYK 到 RGBA 转换、alpha 预乘关联；禁用 OIIO 自动 alpha 关联以保留精度
  - `name()` — 返回文件名（不含路径）
  - `osl_filepath()` — 返回完整文件路径，用于 OSL 纹理缓存
  - `equals()` — 通过比较文件路径判断两个加载器是否引用同一图像

## 核心函数

- `oiio_load_pixels<FileFormat, StorageType>()` — 模板辅助函数，执行实际的 OIIO 像素读取。功能包括：
  - 以翻转扫描线顺序读取图像（从底到顶）
  - 超过 4 通道时截断为 4 通道
  - JPEG CMYK 四通道自动转换为 RGB
  - 根据 `associate_alpha` 参数执行 alpha 预乘

## 依赖关系

- **内部头文件**: `scene/image.h`
- **cpp 额外引用**: `util/image.h`, `util/log.h`, `util/path.h`, `util/unique_ptr.h`
- **外部依赖**: OpenImageIO (`ImageInput`, `ImageSpec`, `TypeDesc`)
- **被引用**: `scene/image.cpp`（在 `add_image()` 中直接构造 `OIIOImageLoader`）

## 实现细节 / 关键算法

1. **元数据检测**: 分析主格式和各通道格式（`spec.channelformats`），取通道大小最大值，自动判断为 float4/float/half4/half/byte4/byte/ushort4/ushort 类型。
2. **Alpha 处理策略**: 使用 `oiio:UnassociatedAlpha` 配置项禁用 OIIO 自动 alpha 关联，自行处理。对 TGA 格式检查 `targa:alpha_type`，对 DDS 和 PSD 格式强制执行关联（作为 OIIO bug 的变通方案）。
3. **CMYK 转换**: 检测 JPEG 格式 4 通道图像，自动执行 CMYK->RGB 转换：`R = (1-C)(1-K)`。
4. **扫描线翻转**: 读取时使用负步长 `-scanlinesize` 翻转 Y 轴，符合渲染器纹理坐标约定。

## 关联文件

- `scene/image.h/.cpp` — 图像管理系统（基类 `ImageLoader` 定义）
- `util/image.h` — 图像工具函数（类型转换、乘法等）
- `util/path.h` — 路径工具函数（`path_exists`, `path_is_directory`, `path_filename`）
