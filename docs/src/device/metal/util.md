# util.h / util.mm - Metal 工具函数与设备信息查询

## 概述

本文件提供了 Cycles Metal 后端的基础工具函数和设备信息查询能力。`MetalInfo` 结构体封装了设备枚举、GPU 架构识别、核心数查询等静态方法。此外，本文件还实现了 GPU 地址获取机制（`GPUAddressHelper`），为无绑定（bindless）纹理和加速结构资源提供 GPU 端地址解析，兼容 macOS 13.0 之前不支持 `gpuAddress` API 的旧系统。

## 类与结构体

### MetalInfo
- **功能**: 提供 Metal 设备相关的静态查询方法
- **关键方法**:
  - `get_usable_devices()` — 枚举并返回所有可用 Metal 设备列表。仅包含 macOS 12.2+ 上的 Apple Silicon GPU（排除 Intel/AMD，要求统一内存）。结果全局缓存，仅首次调用时枚举。
  - `get_apple_gpu_core_count(device)` — 通过 IOKit 查询 Apple GPU 的核心数量（从设备注册表读取 `gpu-core-count` 属性）
  - `get_apple_gpu_architecture(device)` — 通过设备名称字符串匹配识别 GPU 架构（M1/M2/M2 Big/M3/Unknown）
  - `optimal_sort_partition_elements()` — 返回排序分区的最优元素数，默认 65536。在 M1/M2 GPU 上，分区排序可提升最高 15% 的渲染性能（更好的缓存利用率）。可通过 `CYCLES_METAL_SORT_PARTITION_ELEMENTS` 环境变量覆盖。
  - `get_device_name(device)` — 返回设备名称字符串，附加 GPU 核心数后缀（如 "Apple M2 (GPU - 10 cores)"）

### AppleGPUArchitecture（枚举）
- `APPLE_M1` — M1 系列
- `APPLE_M2` — M2 系列（10 核及以下）
- `APPLE_M2_BIG` — M2 系列（10 核以上，如 M2 Pro/Max/Ultra）
- `APPLE_M3` — M3 系列
- `APPLE_UNKNOWN` — 未知架构（始终位于枚举末尾，使比较运算符自动将未来架构视为最新）

### GPUAddressHelper（内部结构体，定义在 util.mm）
- **功能**: 提供跨 macOS 版本的 GPU 地址获取机制
- **关键成员**:
  - `resource_buffer` (`id<MTLBuffer>`) — 用于间接获取 GPU 地址的辅助缓冲区
  - `address_encoder` (`id<MTLArgumentEncoder>`) — 参数编码器，用于将资源编码为 GPU 地址
- **关键方法**:
  - `init(device)` — 初始化辅助缓冲区和参数编码器（macOS 13.0+ 下无需初始化）
  - `gpuAddress(buffer)` — 获取 `MTLBuffer` 的 GPU 地址
  - `gpuResourceID(texture/accel_struct/ift)` — 获取各类 Metal 资源的 GPU Resource ID

## 核心函数

- `metal_gpu_address_helper_init(device)` — 初始化全局 `GPUAddressHelper` 单例
- `metal_gpuAddress(buffer)` — 获取 Metal 缓冲区的 GPU 地址（uint64_t）
- `metal_gpuResourceID(texture)` — 获取 Metal 纹理的 GPU Resource ID
- `metal_gpuResourceID(accel_struct)` — 获取 Metal 加速结构的 GPU Resource ID
- `metal_gpuResourceID(ift)` — 获取 Metal 相交函数表的 GPU Resource ID

## 依赖关系

- **内部头文件**:
  - `device/metal/device.h` — Metal 设备工厂接口
  - `device/metal/kernel.h` — MetalPipelineType 枚举
  - `device/metal/device_impl.h` — MetalDevice 类型（util.mm 中引用）
  - `device/queue.h` — DeviceQueue 基类
  - `util/thread.h` — 线程工具
  - `util/md5.h`, `util/path.h`, `util/string.h`, `util/time.h`
  - `<IOKit/IOKitLib.h>` — macOS IOKit 框架，用于查询硬件信息
- **被引用**:
  - `device/metal/bvh.mm` — BVH 构建使用 GPU 地址函数
  - `device/metal/device_impl.h` — MetalDevice 引用本头文件
  - `device/metal/queue.h` — MetalDeviceQueue 引用本头文件
  - `device/metal/queue.mm` — 使用 `metal_gpuAddress` 和 `metal_gpuResourceID`

## 实现细节 / 关键算法

1. **GPU 地址获取的兼容性处理**: macOS 13.0 引入了 `MTLBuffer.gpuAddress` 和 `MTLResource.gpuResourceID` 直接 API。对于更早的系统，使用一种巧妙的间接方法：创建一个 `MTLArgumentEncoder`，将目标资源编码到一个 8 字节的辅助缓冲区中，然后从缓冲区内容读取编码后的地址。通过 `CYCLES_USE_TIER2D_BINDLESS` 宏控制是否启用直接 API 路径。

2. **设备过滤策略**: `get_usable_devices()` 通过设备名称排除 Intel 和 AMD GPU（如 "Intel UHD Graphics"），仅保留名称包含 "Apple" 的设备，并验证其是否支持统一内存（`hasUnifiedMemory`）。这确保了只在 Apple Silicon 上运行。

3. **架构识别**: `get_apple_gpu_architecture()` 使用简单的字符串匹配（`strstr`）识别 GPU 架构。M2 系列通过核心数（>10 为 Big 变体）进一步区分。`APPLE_UNKNOWN` 位于枚举末尾，确保未来架构在比较时被视为"最新"，从而获得最新的默认配置。

4. **排序分区优化**: M1/M2 GPU 上将活跃路径按材质排序前先分成 65536 元素的分区，可显著提升缓存命中率，总体渲染加速最高 15%。

## 关联文件

- `src/device/metal/device_impl.h` / `device_impl.mm` — 设备初始化时调用 `metal_gpu_address_helper_init()`
- `src/device/metal/queue.mm` — 内核调度时使用 GPU 地址函数编码资源绑定
- `src/device/metal/bvh.mm` — BVH 构建使用 GPU 地址函数
- `src/device/metal/kernel.mm` — 着色器缓存使用架构信息调优
