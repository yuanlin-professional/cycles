# device.h / device.cpp - CUDA 设备注册与发现入口

## 概述

本文件是 Cycles 渲染器 CUDA 后端的顶层入口模块，提供了 CUDA 设备的初始化、创建、信息枚举以及能力查询四大接口函数。它作为 CUDA 后端与 Cycles 设备管理框架之间的桥梁，被通用设备层调用以完成 CUDA 设备的注册与实例化。所有功能均通过 `WITH_CUDA` 编译宏控制，在未编译 CUDA 支持时提供安全的回退行为。

## 类与结构体

本文件不定义类，仅声明和实现四个命名空间级别的自由函数。

## 核心函数

### `device_cuda_init()`
- **返回值**: `bool`
- **功能**: 初始化 CUDA 运行时环境。在动态加载模式（`WITH_CUDA_DYNLOAD`）下通过 `cuewInit()` 初始化 CUEW 封装库，然后检查是否存在预编译内核（`CUDADevice::have_precompiled_kernels()`）或可用的 CUDA 编译器（`cuewCompilerPath()`）。使用静态局部变量保证只初始化一次。在非动态加载模式下直接返回 `true`。

### `device_cuda_create()`
- **签名**: `unique_ptr<Device> device_cuda_create(const DeviceInfo &info, Stats &stats, Profiler &profiler, bool headless)`
- **功能**: 工厂函数，创建并返回一个 `CUDADevice` 实例。当未编译 CUDA 支持时触发 `LOG_FATAL` 错误。

### `device_cuda_info()`
- **签名**: `void device_cuda_info(vector<DeviceInfo> &devices)`
- **功能**: 枚举系统中所有可用的 CUDA 设备。具体步骤：
  1. 通过 `device_cuda_safe_init()` 安全初始化 CUDA（Windows 上使用 SEH 异常处理防止驱动崩溃）
  2. 遍历所有 CUDA 设备，通过 `cudaSupportsDevice()` 过滤不支持的设备（计算能力 < 5.0）
  3. 填充 `DeviceInfo` 结构体，包括设备类型、NanoVDB 支持、P2P 内存访问能力、PCI 位置 ID
  4. 检测 OpenImageDenoise (OIDN) 去噪器支持
  5. 根据内核执行超时和计算抢占属性区分显示设备与计算设备

### `device_cuda_capabilities()`
- **签名**: `string device_cuda_capabilities()`
- **功能**: 收集并返回所有 CUDA 设备的详细硬件能力信息字符串，包括最大线程数、显存总线宽度、L2 缓存大小、计算能力版本等数十项 `CU_DEVICE_ATTRIBUTE_*` 属性。主要用于调试和诊断。

### `device_cuda_safe_init()`（静态内部函数）
- **功能**: 封装 `cuInit(0)` 的安全调用。在 Windows 平台通过 `__try/__except` 结构化异常处理捕获 CUDA 驱动崩溃，防止损坏的 CUDA 安装导致整个应用程序崩溃。

## 依赖关系

- **内部头文件**:
  - `device/cuda/device.h` — 自身头文件
  - `device/device.h` — 通用设备基类与 `DeviceInfo`
  - `device/cuda/device_impl.h` — `CUDADevice` 具体实现（条件编译）
  - `integrator/denoiser_oidn_gpu.h` — OIDN GPU 去噪器（IWYU pragma keep）
  - `util/log.h`、`util/string.h`、`util/windows.h`
- **被引用**:
  - `src/device/device.cpp` — 通用设备层调用本文件接口进行 CUDA 设备注册
  - `src/device/optix/device.cpp` — OptiX 后端也引用本文件

## 实现细节 / 关键算法

1. **设备支持检测**: 通过 `cudaSupportsDevice()` 函数（定义于 `util.h`）检查计算能力 >= 5.0 (Maxwell 架构及以上)。
2. **显示设备识别**: 同时具有内核执行超时（`CU_DEVICE_ATTRIBUTE_KERNEL_EXEC_TIMEOUT`）且不支持计算抢占（`CU_DEVICE_ATTRIBUTE_COMPUTE_PREEMPTION_SUPPORTED`）的设备被标记为显示设备。Windows 10 17134+ 上对 SM 6.0+ 设备强制假设支持计算抢占，以解决 CUDA 驱动在应用程序配置文件中的已知问题。
3. **设备排序策略**: 显示设备被放置到设备列表末尾，优先使用非显示设备进行渲染。
4. **P2P 访问检测**: 通过 `cuDeviceCanAccessPeer()` 检查每对设备之间是否支持点对点内存访问，用于多 GPU 渲染场景。

## 关联文件

- `src/device/cuda/device_impl.h` / `device_impl.cpp` — `CUDADevice` 类的完整实现
- `src/device/device.h` / `device.cpp` — 通用设备管理框架
- `src/device/cuda/util.h` — CUDA 工具函数与 `cudaSupportsDevice()`
- `src/device/optix/device.cpp` — OptiX 后端基于 CUDA 设备构建
