# kernel.h / kernel.cpp - CUDA 内核函数管理与占用率优化

## 概述

本文件定义了 CUDA 内核函数的封装类和缓存管理机制。`CUDADeviceKernel` 封装单个 CUDA 内核函数及其最优执行参数，`CUDADeviceKernels` 管理所有设备内核的加载和查询。这些类为 Cycles 的波前路径追踪积分器提供高效的内核调度基础，通过 CUDA 占用率 API 自动计算每个内核的最优线程块配置。

## 类与结构体

### `CUDADeviceKernel`
- **功能**: 封装单个 CUDA 内核函数及其推荐执行参数
- **关键成员**:
  - `function` (`CUfunction`) — CUDA 内核函数句柄，默认为 `nullptr`
  - `num_threads_per_block` (`int`) — 每个线程块的推荐线程数（由占用率 API 计算），默认为 `0`
  - `min_blocks` (`int`) — 达到最大占用率的最小线程块数，默认为 `0`

### `CUDADeviceKernels`
- **功能**: 所有 CUDA 设备内核的缓存容器，负责批量加载和按名查询
- **关键成员**:
  - `kernels_` (`CUDADeviceKernel[DEVICE_KERNEL_NUM]`) — 内核数组，以 `DeviceKernel` 枚举值为索引
  - `loaded` (`bool`) — 标记内核是否已完成加载
- **关键方法**:
  - `load(CUDADevice *device)` — 从 CUDA 模块加载所有内核函数
  - `get(DeviceKernel kernel)` — 获取指定内核的 `CUDADeviceKernel` 引用
  - `available(DeviceKernel kernel)` — 检查指定内核是否已成功加载

## 核心函数

### `CUDADeviceKernels::load(CUDADevice *device)`
内核加载流程：
1. 获取设备的 `cuModule`
2. 遍历所有 `DEVICE_KERNEL_NUM` 个内核枚举值
3. 跳过 `DEVICE_KERNEL_INTEGRATOR_MEGAKERNEL`（GPU 不使用巨型内核，该内核仅用于 CPU 后端）
4. 构造函数名称：`"kernel_gpu_" + device_kernel_as_string(kernel)`
5. 通过 `cuModuleGetFunction()` 从模块中获取函数句柄
6. 设置 L1 缓存优先策略（`CU_FUNC_CACHE_PREFER_L1`）
7. 调用 `cuOccupancyMaxPotentialBlockSize()` 计算最优线程块大小和最小线程块数
8. 加载失败的内核记录错误日志

### `CUDADeviceKernels::get(DeviceKernel kernel)`
直接通过枚举值索引返回对应的 `CUDADeviceKernel` 常量引用。时间复杂度 O(1)。

### `CUDADeviceKernels::available(DeviceKernel kernel)`
检查对应内核的 `function` 成员是否非空。

## 依赖关系

- **内部头文件**:
  - `device/kernel.h` — `DeviceKernel` 枚举定义与 `device_kernel_as_string()` 函数
  - `device/cuda/device_impl.h` — `CUDADevice` 类（获取 `cuModule`）
  - CUDA 头文件：`cuew.h`（动态加载模式）或 `<cuda.h>`（链接模式）
- **被引用**:
  - `src/device/cuda/device_impl.h` — `CUDADevice` 类包含 `CUDADeviceKernels kernels` 成员
  - `src/device/cuda/queue.cpp` — `CUDADeviceQueue::enqueue()` 通过 `kernels.get()` 获取内核启动参数

## 实现细节 / 关键算法

1. **占用率优化**: `cuOccupancyMaxPotentialBlockSize()` 是 CUDA 提供的启发式 API，它综合考虑寄存器使用量、共享内存需求和硬件限制，自动计算使 GPU 多处理器占用率最大化的线程块大小。这避免了手动调优线程配置的复杂性。
2. **L1 缓存策略**: 通过 `cuFuncSetCacheConfig(CU_FUNC_CACHE_PREFER_L1)` 为每个内核设置优先使用 L1 缓存而非共享内存的策略。路径追踪内核通常不使用共享内存（除了特定的路径排序内核），因此优先分配 L1 缓存可以提升全局内存访问性能。
3. **命名约定**: 所有 GPU 内核函数遵循 `kernel_gpu_` 前缀命名约定，后接通过 `device_kernel_as_string()` 转换的内核名称字符串。
4. **巨型内核排除**: `DEVICE_KERNEL_INTEGRATOR_MEGAKERNEL` 被跳过是因为 GPU 后端使用波前路径追踪架构，将积分器拆分为多个独立内核分阶段执行，而非使用单一巨型内核。

## 关联文件

- `src/device/cuda/device_impl.h` / `device_impl.cpp` — 持有 `CUDADeviceKernels` 实例，在 `load_kernels()` 中调用 `kernels.load()`
- `src/device/cuda/queue.h` / `queue.cpp` — 内核调度时使用 `kernels.get()` 获取启动参数
- `src/device/kernel.h` — `DeviceKernel` 枚举与通用内核接口
