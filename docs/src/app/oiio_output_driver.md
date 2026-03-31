# oiio_output_driver.h / oiio_output_driver.cpp - 基于 OpenImageIO 的图像输出驱动

## 概述

`oiio_output_driver.h` 和 `oiio_output_driver.cpp` 实现了基于 OpenImageIO（OIIO）库的渲染结果图像输出驱动。该类继承自 Cycles 的 `OutputDriver` 接口，负责将渲染完成的图像数据写入磁盘文件。它是 Cycles 独立渲染器将路径追踪结果保存为 PNG、EXR、TIFF 等常见图像格式的核心输出组件。

## 类与结构体

### OIIOOutputDriver
- **继承**: `OutputDriver`（来自 `session/output_driver.h`）
- **功能**: 将渲染完成的图像块（Tile）数据通过 OpenImageIO 写入指定格式的图像文件
- **关键成员**:
  - `filepath_` (`string`) — 输出图像文件路径，文件扩展名决定输出格式
  - `pass_` (`string`) — 要输出的渲染通道名称（如 `"combined"`）
  - `log_` (`LogFunction`) — 日志回调函数，用于输出状态信息
- **类型别名**:
  - `LogFunction` = `std::function<void(const string &)>` — 日志回调函数类型
- **关键方法**:
  - `OIIOOutputDriver(filepath, pass, log)` — 构造函数，接收输出路径、通道名和日志函数
  - `write_render_tile(const Tile &tile)` — 重写基类方法，将渲染块写入图像文件

## 核心函数

### OIIOOutputDriver::write_render_tile()
- **签名**: `void OIIOOutputDriver::write_render_tile(const Tile &tile)`
- **功能**: 渲染完成回调，将渲染结果写入图像文件。核心流程：
  1. **完整缓冲区检查**: 仅在 `tile.size == tile.full_size` 时写入，跳过中间分块渲染的部分瓦片
  2. **创建 ImageOutput**: 通过 `ImageOutput::create(filepath_)` 根据文件扩展名自动选择图像格式
  3. **配置图像规格**: 创建 4 通道（RGBA）浮点格式的 `ImageSpec`
  4. **读取像素数据**: 调用 `tile.get_pass_pixels(pass_, 4, pixels.data())` 获取指定渲染通道的像素数据
  5. **坐标翻转**: 构造 `ImageBuf` 时通过负步长（negative stride）将 Cycles 的自底向上（bottom-up）像素顺序转换为 OIIO 的自顶向下（top-down）约定
  6. **Gamma 校正**: 通过 `ColorSpaceManager::detect_known_colorspace` 检测输出格式是否需要非线性色彩空间转换。对于 sRGB 格式（如 PNG），应用 gamma=1/2.2 的幂校正
  7. **写入文件**: 以浮点格式写入并关闭文件

## 依赖关系

### 内部头文件
- `session/output_driver.h` — `OutputDriver` 基类定义
- `scene/colorspace.h` — 色彩空间管理（`ColorSpaceManager`）
- `util/image.h` — 图像工具
- `util/string.h` — 字符串工具
- `util/unique_ptr.h` — 智能指针

### 外部库
- `<OpenImageIO/imagebuf.h>` — OIIO 图像缓冲区（`ImageBuf`、`ImageSpec`、`ImageOutput`）
- `<OpenImageIO/imagebufalgo.h>` — OIIO 图像算法（`ImageBufAlgo::pow` 用于 gamma 校正）

### 被引用
- `src/app/cycles_standalone.cpp` — 在 `session_init()` 中创建 `OIIOOutputDriver` 实例并设置到渲染会话

## 实现细节 / 关键算法

1. **底部到顶部的像素翻转**: Cycles 内部以自底向上的顺序存储像素数据，而大多数图像格式采用自顶向下约定。通过将 `ImageBuf` 的数据指针设置为最后一行，并使用负的行步长（`-width * 4 * sizeof(float)`），实现零拷贝的坐标翻转。

2. **自动色彩空间检测**: 使用 `ColorSpaceManager::detect_known_colorspace` 根据输出格式名称（如 "png"、"jpeg"）判断是否需要 sRGB 转换。对于线性格式（如 EXR），不做 gamma 校正；对于非线性格式，应用 gamma = 1/2.2 的近似 sRGB 转换。代码中注释标记了未来应使用 OpenColorIO 视图变换的改进方向。

3. **仅写入完整缓冲区**: 通过 `tile.size == tile.full_size` 条件判断，确保只在整个图像渲染完成后才写入文件，避免分块渲染过程中的重复写入。

## 关联文件

- `src/session/output_driver.h` — `OutputDriver` 基类及 `Tile` 数据结构定义
- `src/app/cycles_standalone.cpp` — 使用此驱动的主程序
- `src/scene/colorspace.h` — 色彩空间检测逻辑
