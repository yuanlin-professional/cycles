# colorspace.h / colorspace.cpp - 色彩空间管理与转换

## 概述

本文件实现了 Cycles 的色彩空间管理系统 `ColorSpaceManager`，负责色彩空间的检测、转换和缓存。系统基于 OpenColorIO (OCIO) 库实现自定义色彩空间转换，同时内置了 sRGB 和 raw（场景线性）两种常用色彩空间的快速路径。所有静态方法组成无状态管理器模式。

## 类与结构体

### ColorSpaceManager
- **功能**: 静态色彩空间管理器，处理色彩空间检测与像素转换
- **关键方法**:
  - `detect_known_colorspace()` — 将用户指定的色彩空间转换为可处理的形式。自动模式下根据文件格式和数据类型推断（浮点文件默认 raw，PNG/JPEG 等默认 sRGB）；对自定义色彩空间通过 OCIO 验证并缓存
  - `colorspace_is_data()` — 判断色彩空间是否为非颜色数据（通过 OCIO 的 `isData()` 检查）
  - `to_scene_linear()` — 模板方法，将指定色彩空间的像素批量转换为场景线性空间。支持 RGBA 和灰度两种模式，处理 alpha 预乘/解预乘
  - `to_scene_linear(processor, pixel, channels)` — 单像素实时转换，用于 OSL 纹理缓存场景
  - `get_processor()` — 获取（或创建缓存的）OCIO 色彩空间处理器
  - `free_memory()` — 释放所有缓存的处理器和色彩空间映射
  - `init_fallback_config()` — 创建回退 OCIO 配置（raw），用于回归测试
- **私有方法**:
  - `is_builtin_colorspace()` — 通过对比 256 个采样值判断色彩空间是否等价于 scene_linear 或 sRGB

## 核心函数

- `processor_apply_pixels_rgba<T, compress_as_srgb>()` — OCIO 批量 RGBA 像素处理，分块（16M 像素）处理大图以控制内存，处理 alpha 解预乘/重预乘
- `processor_apply_pixels_grayscale<T, compress_as_srgb>()` — OCIO 批量灰度像素处理，扩展为 3 通道处理后取平均值
- `cast_to_float4<T>()` / `cast_from_float4<T>()` — 任意存储类型与 float4 之间的转换辅助函数
- `check_invalidate_caches()` — 检查 OCIO 配置是否变化（scene_linear 空间名称），变化时清除所有缓存

## 全局变量

- `u_colorspace_auto` — 自动检测色彩空间标识（空字符串）
- `u_colorspace_raw` — 内建 raw 色彩空间（`"__builtin_raw"`）
- `u_colorspace_srgb` — 内建 sRGB 色彩空间（`"__builtin_srgb"`）

## 依赖关系

- **内部头文件**: `util/param.h`
- **cpp 额外引用**: `util/color.h`, `util/image.h`, `util/log.h`, `util/map.h`, `util/math.h`, `util/thread.h`, `util/vector.h`; 条件编译: `OpenColorIO/OpenColorIO.h`
- **被引用**: `scene/image.h`, `scene/image.cpp`, `scene/shader_nodes.cpp`, `scene/osl.cpp`, `kernel/osl/services.cpp`, `app/oiio_output_driver.cpp`, `test/render_graph_finalize_test.cpp`

## 实现细节 / 关键算法

1. **内建色彩空间检测**: `is_builtin_colorspace()` 对 0-255 范围的 256 个值逐一通过 OCIO 处理器转换，检查：无通道串扰、三原色线性叠加、三通道行为一致。然后分别检测是否与恒等变换（scene_linear）或 sRGB->linear 转换匹配。
2. **缓存机制**: 使用三个线程安全的缓存：
   - `cached_colorspaces` — 色彩空间名称到内建类型的映射
   - `cached_processors` — 色彩空间到 OCIO 处理器的映射
   - `cache_scene_linear_name` — 当前 scene_linear 空间名称（用于失效检测）
3. **Alpha 处理**: RGBA 转换时先解预乘 alpha（除以 alpha），应用色彩变换，再重新预乘。alpha 为 0 或 1 时跳过此步骤以优化性能。
4. **分块处理**: 大图像以 16M 像素为单位分块处理，避免临时缓冲区占用过多内存。
5. **模板实例化**: 显式实例化 `to_scene_linear` 模板用于 `uchar`、`ushort`、`half`、`float` 四种类型。

## 关联文件

- `scene/image.h/.cpp` — 图像管理系统（在图像加载时调用色彩空间转换）
- `scene/shader_nodes.cpp` — 着色器节点（纹理节点的色彩空间处理）
- `kernel/osl/services.cpp` — OSL 运行时色彩空间转换
