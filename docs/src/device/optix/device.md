# device.h / device.cpp - OptiX 设备的注册与工厂入口

## 概述

本文件对提供 OptiX 渲染设备的初始化、枚举和创建功能。`device.h` 声明了三个全局函数，作为 Cycles 设备系统发现和实例化 OptiX 设备的统一入口。`device.cpp` 实现了这些函数，负责调用 OptiX SDK 进行运行时初始化，并将符合条件的 CUDA 设备包装为 OptiX 设备对象。

## 类与结构体

本文件不定义类或结构体。仅涉及前向声明：

- `Device` — Cycles 设备基类
- `DeviceInfo` — 设备信息描述结构
- `Profiler` — 性能分析器
- `Stats` — 统计信息收集器

## 核心函数

### `device_optix_init()`
- **返回值**: `bool`
- **功能**: 初始化 OptiX 运行时环境。首先检查 OptiX 函数表是否已初始化（`g_optixFunctionTable`），若未初始化则先调用 `device_cuda_init()` 初始化 CUDA，再调用 `optixInit()` 加载 OptiX。处理两种错误：`OPTIX_ERROR_UNSUPPORTED_ABI_VERSION`（驱动过旧）和其他通用错误。当编译时未启用 `WITH_OPTIX` 宏时直接返回 `false`。

### `device_optix_info(const vector<DeviceInfo> &cuda_devices, vector<DeviceInfo> &devices)`
- **返回值**: `void`
- **功能**: 枚举可用的 OptiX 设备。遍历所有 CUDA 设备信息，过滤掉计算能力（Compute Capability）低于 5.0 的设备（仅 Maxwell 及更高架构支持 OptiX），将其类型从 `DEVICE_CUDA` 修改为 `DEVICE_OPTIX`，追加 `_OptiX` 后缀到设备 ID，并设置支持的降噪器标志（`DENOISER_OPTIX`，以及在满足条件时添加 `DENOISER_OPENIMAGEDENOISE`）。若编译时启用了 OSL 且 OSL 版本满足要求，还会设置 `has_osl = true`。

### `device_optix_create(const DeviceInfo &info, Stats &stats, Profiler &profiler, bool headless)`
- **返回值**: `unique_ptr<Device>`
- **功能**: 工厂函数，创建并返回一个 `OptiXDevice` 实例。在未启用 `WITH_OPTIX` 时触发 `LOG_FATAL` 并返回 `nullptr`。

## 依赖关系

- **内部头文件**:
  - `device/optix/device.h` — 自身头文件
  - `device/cuda/device.h` — CUDA 设备初始化（`device_cuda_init()`）
  - `device/device.h` — 设备基类定义
  - `device/optix/device_impl.h` — `OptiXDevice` 类定义（条件编译）
  - `integrator/denoiser_oidn_gpu.h` — OpenImageDenoise GPU 降噪器支持
  - `util/unique_ptr.h`、`util/vector.h`、`util/log.h`
- **外部依赖**:
  - `<optix_function_table_definition.h>` — OptiX 函数表定义
  - `<OSL/oslconfig.h>`、`<OSL/oslversion.h>` — OSL 版本检测（条件编译）
- **被引用**:
  - `src/device/device.cpp` — Cycles 主设备管理器调用本文件的函数进行 OptiX 设备注册

## 实现细节 / 关键算法

1. **双重初始化保护**: `device_optix_init()` 通过检查 `g_optixFunctionTable.optixDeviceContextCreate` 是否为 `nullptr` 来避免重复初始化。
2. **CUDA 前置依赖**: OptiX 依赖于 CUDA 运行时，因此在调用 `optixInit()` 之前必须先成功调用 `device_cuda_init()`。
3. **设备枚举策略**: 直接复用已发现的 CUDA 设备列表，通过修改 `DeviceInfo` 的类型字段将其转化为 OptiX 设备，避免重复枚举硬件。
4. **OIDN 版本兼容**: 针对 OIDN 2.3.0 及以上版本和更早版本使用不同的 API 来检测 CUDA 设备是否支持 OpenImageDenoise。

## 关联文件

- `src/device/optix/device_impl.h` / `device_impl.cpp` — `OptiXDevice` 类的完整定义和实现
- `src/device/cuda/device.h` / `device.cpp` — CUDA 设备层，OptiX 设备的基础
- `src/device/device.h` / `device.cpp` — Cycles 通用设备接口
- `src/integrator/denoiser_oidn_gpu.h` — GPU 端 OIDN 降噪器
