# queue.h / queue.cpp - oneAPI 设备命令队列

## 概述

本文件定义并实现了 `OneapiDeviceQueue` 类，是 Cycles 渲染器 oneAPI 后端的设备命令队列。它继承自 `DeviceQueue`，负责管理路径追踪波前（wavefront）内核的调度执行、工作负载并发度计算、设备内存操作以及图形互操作对象的创建。该队列作为 Cycles 积分器（integrator）与底层 oneAPI 设备之间的执行调度层。

## 类与结构体

### `OneapiDeviceQueue`
- **继承**: `DeviceQueue`（来自 `device/queue.h`）
- **功能**: 封装 oneAPI 设备上的内核执行队列，为波前路径追踪提供内核派发、同步和内存操作接口
- **关键成员**:
  - `oneapi_device_` (`OneapiDevice*`) — 关联的 oneAPI 设备实例指针
  - `kernel_context_` (`unique_ptr<KernelContext>`) — 内核执行上下文，包含 SYCL 队列指针和内核全局数据指针
- **关键方法**:
  - `enqueue()` — 派发内核到设备执行
  - `synchronize()` — 等待队列中所有操作完成
  - `num_concurrent_states()` — 计算并发路径状态数
  - `num_sort_partitions()` — 计算排序分区数
  - `graphics_interop_create()` — 创建图形互操作对象

### `KernelExecutionInfo` (文件作用域结构体)
- **功能**: 记录内核执行统计信息（用于性能分析）
- **成员**:
  - `elapsed_summary` (`double`) — 累计执行时间
  - `enqueue_count` (`int`) — 入队次数

## 核心函数

### `OneapiDeviceQueue(device)` (构造函数)
- 调用基类 `DeviceQueue(device)` 构造函数
- 保存 `OneapiDevice*` 指针到 `oneapi_device_` 成员

### `num_concurrent_states(state_size)`
- **公式**: `4 * num_concurrent_busy_states(state_size)`
- **功能**: 返回总并发路径状态数（包含空闲和忙碌状态），用于路径追踪积分器分配状态缓冲区大小

### `num_concurrent_busy_states(state_size)`
- **公式**: `4 * max(8 * max_num_threads, 65536)`
- 其中 `max_num_threads = get_num_multiprocessors() * get_max_num_threads_per_multiprocessor()`
- **功能**: 返回忙碌（活跃执行中）的并发路径状态数。基于 GPU 的 EU 数量和每 EU 线程数计算，保证至少 65536 个状态

### `num_sort_partitions(max_num_paths, max_scene_shaders)`
- **分区元素数选择**:
  - 每 EU 线程数 >= 128: 分区大小 65536
  - 否则: 分区大小 8192
- **公式**: `max(max_num_paths / sort_partition_elements, 1)`
- **功能**: 计算波前路径排序的分区数。Intel GPU 上的本地排序方案不依赖着色器数量，仅基于路径数量分区

### `init_execution()`
- 调用 `oneapi_device_->load_texture_info()` 刷新纹理信息
- 获取设备的 SYCL 队列和内核全局数据指针
- 构造 `KernelContext` 对象并填充队列和全局数据指针
- 调用 `debug_init_execution()` 进行调试初始化

### `enqueue(kernel, kernel_work_size, args)`
- **错误检查**: 设备已有错误时直接返回 `false`
- **纹理更新**: 检查纹理信息是否因内存迁移到主机而需要更新，如需要则先同步
- **工作组大小调整**: 通过 `get_adjusted_global_and_local_sizes()` 获取适合内核类型的全局和本地工作组大小
- **设置场景参数**: 将 `scene_max_shaders` 写入内核上下文
- **内核派发**: 调用 `oneapi_device_->enqueue_kernel()` 执行实际的 SYCL 内核提交
- **错误处理**: 执行失败时通过 `set_error()` 记录包含内核名称和运行时异常信息的错误
- 调用 `debug_enqueue_begin/end()` 进行调试跟踪

### `synchronize()`
- 调用 `oneapi_device_->queue_synchronize()` 等待 SYCL 队列完成所有挂起操作
- 失败时记录错误信息
- 调用 `debug_synchronize()` 进行调试跟踪
- 返回值同时考虑同步结果和设备已有错误状态

### `zero_to_device(mem)`
- 委托给 `oneapi_device_->mem_zero()` 将设备内存清零

### `copy_to_device(mem)`
- 委托给 `oneapi_device_->mem_copy_to()` 将主机数据拷贝到设备

### `copy_from_device(mem)`
- 委托给 `oneapi_device_->mem_copy_from()` 将设备数据拷贝回主机

### `supports_local_atomic_sort()`
- 始终返回 `true`，表示 oneAPI 设备支持基于本地原子操作的排序（用于波前路径排序优化）

### `graphics_interop_create()` (条件编译 `SYCL_LINEAR_MEMORY_INTEROP_AVAILABLE`)
- 创建并返回 `OneapiDeviceGraphicsInterop` 实例，用于 Vulkan 图形互操作

## 依赖关系

### 内部头文件
- `device/oneapi/queue.h` — 本文件自身的头文件声明
- `device/memory.h` — `device_memory` 定义
- `device/queue.h` — 基类 `DeviceQueue` 定义
- `device/oneapi/device_impl.h` — `OneapiDevice` 完整定义（实现文件中引用）
- `device/oneapi/graphics_interop.h` — `OneapiDeviceGraphicsInterop` 定义（实现文件中引用）
- `kernel/device/oneapi/kernel.h` — `KernelContext`、`DeviceKernel` 等定义
- `util/log.h`, `util/unique_ptr.h`

### 被引用
- `src/device/oneapi/device_impl.h` — `OneapiDevice` 头文件中引用队列头文件
- `src/device/oneapi/graphics_interop.h` — 图形互操作头文件中引用队列头文件
- `src/device/oneapi/graphics_interop.cpp` — 图形互操作实现中使用队列

## 实现细节 / 关键算法

1. **波前并发度计算**: oneAPI 的并发状态数采用 `4 * max(8 * max_threads, 65536)` 的公式，其中外层 4 倍用于总状态数与忙碌状态数的比值。`max_threads` 基于 Intel GPU 的 EU 数量乘以每 EU 硬件线程数，确保充分利用 GPU 并行能力。

2. **本地排序分区策略**: Intel GPU 上的波前路径排序始终使用本地排序方案（`supports_local_atomic_sort()` 返回 `true`），分区大小根据设备的 SIMD 能力自适应：大 EU 线程数设备使用 65536 大分区以减少分区数，小 EU 线程数设备使用 8192 小分区以提高排序局部性。

3. **纹理信息延迟更新**: `enqueue()` 在每次内核派发前检查纹理信息是否需要更新。当内存管理器将纹理数据从设备迁移到主机时，纹理指针会变化，需要在下次内核执行前同步更新。若更新发生，必须先同步队列以确保数据一致性。

4. **内核上下文生命周期**: `KernelContext` 在 `init_execution()` 中创建，持有 SYCL 队列和内核全局数据的原始指针。它在整个执行批次期间保持有效，并在下一次 `init_execution()` 调用时被新实例替换。

5. **内存操作委托模式**: `zero_to_device`、`copy_to_device`、`copy_from_device` 直接委托给 `OneapiDevice` 的对应方法，而非直接操作 SYCL 队列。这确保内存操作经过设备层的错误检查和类型分发逻辑。

## 关联文件
- `src/device/oneapi/device_impl.h` / `device_impl.cpp` — 被委托执行实际的内核派发和内存操作
- `src/device/oneapi/graphics_interop.h` / `graphics_interop.cpp` — 队列创建的图形互操作对象
- `src/device/queue.h` — 跨后端的设备队列基类
- `src/kernel/device/oneapi/kernel.h` — 内核函数和 `KernelContext` 定义
