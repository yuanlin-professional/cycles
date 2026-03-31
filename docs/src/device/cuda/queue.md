# queue.h / queue.cpp - CUDA 设备命令队列

## 概述

本文件实现了 `CUDADeviceQueue` 类，是 Cycles 渲染器 CUDA 后端的异步命令队列。它封装了 CUDA 流（`CUstream`）的创建、内核启动、内存传输以及同步操作，是波前路径追踪积分器调度 GPU 计算任务的核心执行通道。命令队列还负责管理图形互操作实例的创建，支持视口实时渲染预览。

## 类与结构体

### `CUDADeviceQueue`
- **继承**: `DeviceQueue`（定义于 `device/queue.h`）
- **功能**: 管理 CUDA 流上的异步内核执行、内存传输和同步
- **关键成员**:
  - `cuda_device_` (`CUDADevice*`) — 关联的 CUDA 设备指针
  - `cuda_stream_` (`CUstream`) — CUDA 异步流句柄
- **关键方法**:
  - `num_concurrent_states()` — 计算并发路径状态数量
  - `num_concurrent_busy_states()` — 计算并发忙碌状态数量
  - `init_execution()` — 初始化执行环境，同步纹理和内存
  - `enqueue()` — 向队列提交内核执行任务
  - `synchronize()` — 等待队列中所有任务完成
  - `zero_to_device()` — 异步将设备内存清零
  - `copy_to_device()` — 异步从主机拷贝数据到设备
  - `copy_from_device()` — 异步从设备拷贝数据到主机
  - `stream()` — 返回底层 CUDA 流句柄（虚函数，供子类覆盖）
  - `graphics_interop_create()` — 创建 `CUDADeviceGraphicsInterop` 实例

## 核心函数

### `CUDADeviceQueue::CUDADeviceQueue(CUDADevice *device)`
构造函数通过 `cuStreamCreate()` 创建非阻塞 CUDA 流（`CU_STREAM_NON_BLOCKING` 标志），确保该流不会与默认流隐式同步。

### `num_concurrent_states(const size_t state_size)`
计算积分器可以同时处理的路径状态数量：
1. 基础值为 `max(多处理器数 * 每多处理器最大线程数, 65536) * 16`
2. 支持通过环境变量 `CYCLES_CONCURRENT_STATES_FACTOR` 调整倍率
3. 记录日志输出最终状态数和预估内存使用量

### `num_concurrent_busy_states(const size_t state_size)`
计算"忙碌"状态的数量，固定为 `4 * max_num_threads`（多处理器数 * 每多处理器最大线程数），最小值 65536。忙碌状态是指正在被 GPU 线程实际处理的路径。

### `init_execution()`
执行前初始化：
1. 进入 CUDA 上下文作用域
2. 调用 `load_texture_info()` 确保纹理信息已上传
3. 调用 `cuCtxSynchronize()` 全局同步（确保所有未完成的内存传输完成）
4. 调用 `debug_init_execution()` 初始化调试追踪

### `enqueue(DeviceKernel kernel, const int work_size, const DeviceKernelArguments &args)`
内核提交流程：
1. 检查设备是否有错误
2. 调用 `debug_enqueue_begin()` 开始调试记录
3. 进入 CUDA 上下文
4. 再次检查并更新纹理信息（积分器内存分配可能导致纹理迁移到主机）
5. 获取对应的 `CUDADeviceKernel`，计算线程块数 = `ceil(work_size / num_threads_per_block)`
6. 为路径索引排序/压缩相关内核分配共享内存（`(num_threads_per_block + 1) * sizeof(int)`）
7. 调用 `cuLaunchKernel()` 异步启动内核

### `synchronize()`
调用 `cuStreamSynchronize()` 等待当前流上所有操作完成，并检查设备错误状态。

### 内存传输方法
- `zero_to_device()`: 使用 `cuMemsetD8Async()` 异步清零，按需自动分配
- `copy_to_device()`: 使用 `cuMemcpyHtoDAsync()` 异步主机到设备拷贝，按需自动分配
- `copy_from_device()`: 使用 `cuMemcpyDtoHAsync()` 异步设备到主机拷贝

### `assert_success(CUresult result, const char *operation)`
统一的错误处理函数，将 CUDA 错误码转换为可读的错误消息，并通过 `cuda_device_->set_error()` 报告，包含操作名称和当前活跃内核信息。

## 依赖关系

- **内部头文件**:
  - `device/queue.h` — 基类 `DeviceQueue`
  - `device/memory.h` — `device_memory` 类型
  - `device/cuda/util.h` — `CUDAContextScope`、错误检查宏
  - `device/cuda/device_impl.h` — `CUDADevice` 类
  - `device/cuda/graphics_interop.h` — `CUDADeviceGraphicsInterop` 类
  - `device/cuda/kernel.h` — `CUDADeviceKernel`、`CUDADeviceKernels`
- **被引用**:
  - `src/device/cuda/device_impl.h` — `CUDADevice::gpu_queue_create()` 创建本类实例
  - `src/device/cuda/device_impl.cpp` — `reserve_local_memory()` 中使用 `CUDADeviceQueue` 预分配内存
  - `src/device/optix/queue.h` — `OptiXDeviceQueue` 可能引用本类（OptiX 队列基于 CUDA 流）

## 实现细节 / 关键算法

1. **非阻塞流**: 使用 `CU_STREAM_NON_BLOCKING` 创建流，确保内核启动不会被默认流上的操作隐式阻塞，允许真正的异步执行。
2. **共享内存计算**: 路径排序/压缩内核（如 `INTEGRATOR_QUEUED_PATHS_ARRAY`、`INTEGRATOR_COMPACT_PATHS_ARRAY` 等）需要额外的共享内存用于并行前缀和（parallel prefix sum）操作。共享内存大小为 `(线程块大小 + 1) * sizeof(int)`，参见 `parall_active_index.h`。
3. **按需内存分配**: `zero_to_device()` 和 `copy_to_device()` 在设备指针为 0 时自动调用 `cuda_device_->mem_alloc()` 分配内存，简化了调用方的使用逻辑。
4. **并发状态数计算**: 并发状态数远大于 GPU 物理线程数（16 倍），这是为了确保波前路径追踪积分器有足够的路径可以调度，隐藏内存延迟并保持 GPU 高占用率。
5. **纹理信息热更新**: `enqueue()` 中在每次内核启动前检查纹理信息变化，因为积分器的内存分配可能触发纹理从设备内存迁移到主机映射内存。

## 关联文件

- `src/device/cuda/device_impl.h` / `device_impl.cpp` — 创建队列实例，队列引用设备进行内存管理
- `src/device/cuda/kernel.h` / `kernel.cpp` — 队列使用内核缓存获取启动参数
- `src/device/cuda/graphics_interop.h` / `graphics_interop.cpp` — 队列创建图形互操作实例
- `src/device/cuda/util.h` — 上下文管理和错误检查
- `src/device/queue.h` — 基类定义
