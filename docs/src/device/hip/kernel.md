# kernel.h / kernel.cpp - HIP 内核函数管理

## 概述

本文件实现了 HIP 内核函数的加载与缓存管理。`HIPDeviceKernel` 封装单个 GPU 内核函数及其占用率信息，`HIPDeviceKernels` 作为所有设备内核的缓存容器，负责从已加载的 HIP 模块中批量提取内核函数句柄并计算最优线程块配置。这两个类被 `HIPDevice` 持有，是波前路径追踪器将工作分派到 GPU 的核心桥梁。

## 类与结构体

### `HIPDeviceKernel`
- **功能**: 封装单个 HIP 内核函数的句柄和最优启动参数
- **关键成员**:
  - `hipFunction_t function` — HIP 内核函数句柄，初始值为 `nullptr`
  - `int num_threads_per_block` — 最优每线程块线程数（由占用率计算器确定）
  - `int min_blocks` — 达到最大占用率所需的最小线程块数

### `HIPDeviceKernels`
- **功能**: 缓存所有 `DeviceKernel` 枚举值对应的 HIP 内核函数
- **关键成员**:
  - `HIPDeviceKernel kernels_[DEVICE_KERNEL_NUM]` — 内核函数数组，按 `DeviceKernel` 枚举索引
  - `bool loaded` — 标记内核是否已加载完成
- **关键方法**:
  - `load()` — 从 HIP 模块批量加载所有内核函数
  - `get()` — 按枚举值获取内核信息
  - `available()` — 检查某内核是否可用

## 核心函数

### `HIPDeviceKernels::load()`
- **参数**: `HIPDevice *device`
- **功能**: 遍历所有 `DeviceKernel` 枚举值（共 `DEVICE_KERNEL_NUM` 个），从设备的 `hipModule` 中加载对应内核函数。具体步骤：
  1. 跳过 `DEVICE_KERNEL_INTEGRATOR_MEGAKERNEL`（GPU 不使用超级内核，使用波前架构的分离内核）
  2. 拼接函数名：`"kernel_gpu_" + device_kernel_as_string(i)`
  3. 调用 `hipModuleGetFunction()` 获取函数句柄
  4. 对成功加载的函数：
     - 设置 L1 缓存优先 (`hipFuncCachePreferL1`)
     - 调用 `hipModuleOccupancyMaxPotentialBlockSize()` 计算最优线程块大小和最小块数
  5. 加载失败时输出错误日志
  6. 所有内核处理完毕后设置 `loaded = true`

### `HIPDeviceKernels::get()`
- **参数**: `DeviceKernel kernel`
- **返回值**: `const HIPDeviceKernel&`
- **功能**: 通过枚举值索引返回对应的内核信息，供队列在分派时查询线程块配置

### `HIPDeviceKernels::available()`
- **参数**: `DeviceKernel kernel`
- **返回值**: `bool`
- **功能**: 检查指定内核的函数句柄是否非空，判断该内核是否可用

## 依赖关系
- **内部头文件**:
  - `device/kernel.h` — `DeviceKernel` 枚举定义、`device_kernel_as_string()` 函数
  - `device/hip/device_impl.h` — `HIPDevice` 类（访问 `hipModule` 成员）
  - `hipew.h` — HIP API 动态加载包装（条件编译 `WITH_HIP_DYNLOAD`）
- **被引用**: `src/device/hip/device_impl.h`（`HIPDevice` 持有 `HIPDeviceKernels kernels` 成员）、`src/device/hip/queue.cpp`（队列分派时查询内核信息）

## 实现细节 / 关键算法

- **内核命名约定**: HIP 内核函数的命名规则为 `kernel_gpu_` 前缀加上 `DeviceKernel` 枚举的字符串表示。例如 `DEVICE_KERNEL_INTEGRATOR_SHADE_SURFACE` 对应函数名 `kernel_gpu_integrator_shade_surface`。
- **占用率优化**: 使用 `hipModuleOccupancyMaxPotentialBlockSize()` 自动计算最优的每块线程数和最小块数，以最大化 GPU 多处理器占用率。这避免了手动调优线程块大小的需要。
- **L1 缓存策略**: 所有内核统一设置 `hipFuncCachePreferL1`，优先使用 L1 缓存而非共享内存，适合路径追踪工作负载中大量全局内存访问的特点。
- **超级内核排除**: GPU 路径追踪使用波前（Wavefront）架构，将积分器拆分为多个小内核依次调度，而非单一超级内核，因此 `DEVICE_KERNEL_INTEGRATOR_MEGAKERNEL` 被显式跳过。

## 关联文件
- `src/device/hip/device_impl.h` / `device_impl.cpp` — `HIPDevice::load_kernels()` 调用 `kernels.load(this)`
- `src/device/hip/queue.cpp` — `HIPDeviceQueue::enqueue()` 调用 `kernels.get(kernel)` 获取启动参数
- `src/device/kernel.h` — `DeviceKernel` 枚举与 `DEVICE_KERNEL_NUM` 常量定义
- `src/kernel/device/hip/` — GPU 端内核源码（被编译到 fatbin 中）
