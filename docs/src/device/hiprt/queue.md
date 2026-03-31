# queue.h / queue.cpp - HIPRT 设备队列：支持硬件光线追踪的 GPU 内核执行队列

## 概述

本文件实现了 HIPRT 设备专用的 GPU 命令队列。`HIPRTDeviceQueue` 类继承自 `HIPDeviceQueue`，在标准 HIP 内核执行基础上增加了对 HIPRT 硬件光线追踪内核的特殊处理。当内核包含光线相交操作时，队列会自动创建 HIPRT 全局遍历栈缓冲区，并将其作为额外参数注入内核调用，使用 HIPRT 专用的线程分组大小来启动内核。

## 类与结构体

### HIPRTDeviceQueue

- **继承**: `HIPDeviceQueue`（位于 `device/hip/queue.h`）
- **功能**: 负责将路径追踪内核提交到 GPU 执行。对于需要光线相交的内核，自动管理 HIPRT 全局遍历栈并调整内核启动参数；对于不涉及光线相交的内核，直接委托给父类 `HIPDeviceQueue` 处理。
- **关键成员**:
  - `hiprt_device_` (`HIPRTDevice*`) — 指向所属 HIPRT 设备的指针，用于访问 HIPRT 上下文、全局栈缓冲区等资源
- **关键方法**:
  - `HIPRTDeviceQueue(HIPRTDevice *device)` — 构造函数，将 `HIPRTDevice` 指针同时传递给父类 `HIPDeviceQueue`（通过强制转换为 `HIPDevice*`）并保存为 `hiprt_device_`
  - `~HIPRTDeviceQueue()` — 默认析构函数
  - `enqueue(DeviceKernel kernel, const int work_size, const DeviceKernelArguments &args)` — 核心方法，将内核提交到 GPU 执行

## 核心函数

### `enqueue(DeviceKernel kernel, const int work_size, const DeviceKernelArguments &args)`

内核排队执行的核心逻辑，处理流程如下：

1. **错误检查**：若设备已处于错误状态，直接返回 `false`。

2. **内核分类**：调用 `device_kernel_has_intersection(kernel)` 判断内核是否包含光线相交操作。若不包含（如纯计算内核），直接委托给 `HIPDeviceQueue::enqueue()` 处理，使用标准 HIP 执行路径。

3. **调试追踪**：调用 `debug_enqueue_begin()` 记录内核执行起始信息。

4. **HIP 上下文设置**：通过 `HIPContextScope` RAII 对象确保 HIP 上下文在当前线程激活。

5. **获取内核函数**：从设备的内核管理器中获取已编译的 HIP 内核函数句柄。

6. **全局栈缓冲区创建**（惰性初始化）：
   - 若 `global_stack_buffer.stackData` 为空，表示尚未创建
   - 调用 `num_concurrent_states(0)` 获取最大并发路径数
   - 配置栈参数：全局栈类型（`hiprtStackTypeGlobal`）、整数条目类型（`hiprtStackEntryTypeInteger`）、线程栈大小（`HIPRT_THREAD_STACK_SIZE`）、最大路径数
   - 调用 `hiprtCreateGlobalStackBuffer()` 创建栈缓冲区

7. **参数注入**：复制原始内核参数，追加 HIPRT 全局栈缓冲区作为 `HIPRT_GLOBAL_STACK` 类型的额外参数。

8. **内核启动**：
   - 线程块大小：`HIPRT_THREAD_GROUP_SIZE`（HIPRT 专用的线程分组常量）
   - 网格大小：`divide_up(work_size, num_threads_per_block)`（向上取整）
   - 共享内存：0 字节
   - 通过 `hipModuleLaunchKernel()` 启动内核

9. **调试追踪**：调用 `debug_enqueue_end()` 记录内核执行结束信息。

10. **返回结果**：检查设备是否出现错误，返回执行状态。

## 依赖关系

### 内部头文件（queue.h）
- `device/memory.h` — 设备内存管理基础设施
- `device/queue.h` — 设备队列基类 `DeviceQueue`
- `device/hip/queue.h` — 父类 `HIPDeviceQueue` 定义

### 内部头文件（queue.cpp）
- `device/hiprt/queue.h` — 本头文件
- `device/hip/graphics_interop.h` — HIP 图形互操作
- `device/hip/kernel.h` — HIP 内核管理，提供 `HIPDeviceKernel` 结构体
- `device/hiprt/device_impl.h` — `HIPRTDevice` 类定义，访问 HIPRT 上下文和全局栈
- `kernel/device/hiprt/globals.h` — HIPRT 内核全局参数定义，包含 `HIPRT_THREAD_STACK_SIZE` 和 `HIPRT_THREAD_GROUP_SIZE`

### 被引用
- `src/device/hiprt/device_impl.h` — 头文件中 `#include "device/hiprt/queue.h"`
- `src/device/CMakeLists.txt` — 构建系统配置

## 实现细节 / 关键算法

### 内核执行路径分离
`enqueue()` 方法实现了两条执行路径的智能分离：
- **非相交内核**（如积分器状态初始化、薄膜过滤等）：直接通过 `HIPDeviceQueue::enqueue()` 以标准 HIP 方式执行，无需 HIPRT 栈开销。
- **相交内核**（如路径追踪积分器、阴影光线等）：使用 HIPRT 专用路径，注入全局遍历栈参数，使用 HIPRT 优化的线程分组大小。

这一设计确保仅在需要硬件光线追踪时才引入 HIPRT 相关开销。

### 全局遍历栈缓冲区
HIPRT 的光线遍历算法需要一个全局栈来追踪 BVH 节点遍历状态。该栈在首次执行相交内核时惰性创建，此后复用。栈的关键参数包括：
- **栈类型**：`hiprtStackTypeGlobal`，即全局共享栈，由所有线程共用
- **条目类型**：`hiprtStackEntryTypeInteger`，栈条目为整数（BVH 节点索引）
- **线程栈大小**：`HIPRT_THREAD_STACK_SIZE`，每个线程的栈深度
- **最大路径数**：由 `num_concurrent_states(0)` 决定，对应 GPU 可并发执行的路径数

### 线程分组策略
HIPRT 相交内核使用专用的 `HIPRT_THREAD_GROUP_SIZE` 作为线程块大小，而非标准 HIP 内核的默认线程块大小。这个值经过优化以匹配 HIPRT 内部的遍历算法和 AMD GPU 的波前（wavefront）调度特性。

### 参数传递机制
内核参数通过复制原始 `DeviceKernelArguments` 并追加 HIPRT 全局栈缓冲区来构造。追加操作使用 `DeviceKernelArguments::HIPRT_GLOBAL_STACK` 类型标识符，确保 GPU 端内核函数能正确解析参数布局。

## 关联文件

- `src/device/hip/queue.h` / `src/device/hip/queue.cpp` — 父类 HIP 设备队列实现，处理非相交内核的执行
- `src/device/hiprt/device_impl.h` / `src/device/hiprt/device_impl.cpp` — HIPRT 设备实现，管理 HIPRT 上下文和全局栈缓冲区
- `src/kernel/device/hiprt/globals.h` — 定义 `KernelParamsHIPRT`、`HIPRT_THREAD_STACK_SIZE`、`HIPRT_THREAD_GROUP_SIZE` 等常量
- `src/device/hip/kernel.h` — `HIPDeviceKernel` 结构体和内核加载管理
