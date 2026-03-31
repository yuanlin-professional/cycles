# openimagedenoise.h - OpenImageDenoise 降噪器集成封装

## 概述
本文件封装了 Intel OpenImageDenoise（OIDN）库的头文件引入和平台支持检测。OIDN 是一个基于深度学习的图像降噪器，用于在渲染过程中或渲染后对蒙特卡洛噪声进行降噪处理。本文件通过编译条件 `WITH_OPENIMAGEDENOISE` 控制是否启用，并提供了运行时 CPU 能力检测函数。

## 类与结构体
无自定义类或结构体。当 `WITH_OPENIMAGEDENOISE` 启用时，导出 `<OpenImageDenoise/oidn.hpp>` 中的所有 OIDN API。

## 核心函数

### `bool openimagedenoise_supported()`
检测当前平台是否支持 OpenImageDenoise 降噪器：

| 平台 | 检测逻辑 |
|------|----------|
| macOS（所有架构） | 始终支持（通过 Accelerate 框架 BNNS） |
| ARM64（Windows/Linux） | 始终支持（OIDN 2.2+ 支持 ARM64） |
| x86_64 | 需要 SSE4.2 指令集支持（通过 `system_cpu_support_sse42()` 检测） |
| 未编译 OIDN | 返回 `false` |

## 依赖关系
- **内部头文件**: `util/system.h`（CPU 能力检测）
- **外部依赖**: `<OpenImageDenoise/oidn.hpp>`（条件编译）
- **编译条件**: `WITH_OPENIMAGEDENOISE`
- **被引用**: `integrator/denoiser_oidn.cpp`, `session/session.cpp`

## 关联文件
- `util/system.h` - `system_cpu_support_sse42()` CPU 特性检测
- `integrator/denoiser_oidn.cpp` - OIDN 降噪器的具体实现
- `integrator/denoiser.h` - 降噪器抽象接口
