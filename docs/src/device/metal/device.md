# device.h / device.mm - Metal 设备后端的注册与工厂接口

## 概述

本文件定义了 Cycles 渲染器 Metal 设备后端的全局入口函数，负责设备的初始化、退出、信息枚举、实例创建和能力查询。这些函数由通用设备管理层 (`device/device.cpp`) 调用，是 Metal 后端与上层设备框架之间的桥梁。

## 类与结构体

本文件不定义类，仅声明和实现全局函数。

## 核心函数

- `device_metal_init()` — 初始化 Metal 设备后端，当前实现始终返回 `true`
- `device_metal_exit()` — 退出 Metal 设备后端，调用 `MetalDeviceKernels::static_deinitialize()` 释放所有着色器缓存的静态资源
- `device_metal_create(info, stats, profiler, headless)` — 工厂函数，创建并返回一个 `MetalDevice` 实例（`unique_ptr<Device>`）
- `device_metal_info(devices)` — 枚举系统中所有可用的 Metal 设备，填充 `DeviceInfo` 列表。对每个设备设置以下关键属性：
  - 设备类型设为 `DEVICE_METAL`
  - 检测 OpenImageDenoise (OIDN) 支持
  - 启用 NanoVDB 支持
  - 检测 MNEE（Manifold Next Event Estimation）支持（需 macOS 13+）
  - 检测硬件光线追踪支持（需 macOS 14+ 且设备支持 `supportsRaytracing`）
  - M3 及以上架构默认启用 MetalRT
- `device_metal_capabilities()` — 返回所有 Metal 设备的能力描述字符串，用于日志和调试

## 依赖关系

- **内部头文件**:
  - `device/device.h` — 通用设备接口定义
  - `device/metal/device.h`（自身头文件）
  - `device/metal/device_impl.h` — MetalDevice 实现类
  - `integrator/denoiser_oidn_gpu.h` — OIDN GPU 降噪器
  - `util/string.h`, `util/unique_ptr.h`, `util/vector.h` — 工具库
  - `util/debug.h`, `util/set.h`, `util/system.h`
- **被引用**:
  - `device/device.cpp` — 通用设备管理层调用本文件的全局函数进行 Metal 设备注册
  - `device/metal/device_impl.h` — 引用本头文件
  - `device/metal/device_impl.mm` — 引用本头文件
  - `device/metal/util.h` — 引用本头文件

## 实现细节 / 关键算法

1. **设备枚举**: `device_metal_info()` 调用 `MetalInfo::get_usable_devices()` 获取所有可用设备，为每个设备生成唯一 ID（格式为 `METAL_<设备名>`），并在 ID 重复时追加序号后缀。

2. **条件编译**: 当 `WITH_METAL` 未定义时，所有函数提供空的占位实现，确保非 macOS 平台编译不会出错。

3. **设备能力检测**: 通过 `@available` 运行时检查和编译时宏（如 `MAC_OS_VERSION_14_0`）确保各功能仅在支持的 macOS 版本上启用。

## 关联文件

- `src/device/device.cpp` — 调用方，统一管理所有设备后端的注册
- `src/device/metal/device_impl.h` / `device_impl.mm` — `MetalDevice` 类的完整实现
- `src/device/metal/util.h` / `util.mm` — `MetalInfo` 工具类
