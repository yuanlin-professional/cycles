# queue.h / queue.cpp - HIP 设备命令队列

## 概述

本文件实现了 HIP 后端的设备命令队列 `HIPDeviceQueue`，继承自 `DeviceQueue` 基类。队列封装了一个 HIP 流（`hipStream_t`），负责内核分派、流同步、异步内存操作以及图形互操作对象创建。它是波前路径追踪器与 GPU 硬件之间的调度接口，积分器通过该队列将各阶段内核依次提交到 GPU 执行。

## 类与结构体

### `HIPDeviceQueue`
- **继承**: `DeviceQueue`（`device/queue.h` 中定义的设备队列基类）
- **功能**: 封装 HIP 流，管理内核启动、内存传输和同步操作
- **关键成员**:
  - `HIPDevice *hip_device_` — 关联的 HIP 设备指针
  - `hipStream_t hip_stream_` — HIP 异步流句柄
- **关键方法**:
  - `enqueue()` — 向 GPU 提交内核执行
  - `synchronize()` — 等待流中所有操作完成
  - `init_execution()` — 在执行任务前同步纹理和内存
  - `zero_to_device()` / `copy_to_device()` / `copy_from_device()` — 异步内存操作
  - `stream()` — 获取底层 HIP 流句柄（虚函数，可被子类覆盖）
  - `graphics_interop_create()` — 创建图形互操作对象
  - `num_concurrent_states()` / `num_concurrent_busy_states()` — 计算并发状态数

## 核心函数

### `HIPDeviceQueue::HIPDeviceQueue()`
- 调用 `hipStreamCreateWithFlags()` 创建非阻塞流（`hipStreamNonBlocking`），使其不会被默认流阻塞

### `HIPDeviceQueue::~HIPDeviceQueue()`
- 调用 `hipStreamDestroy()` 销毁流

### `HIPDeviceQueue::num_concurrent_states()`
- **参数**: `size_t state_size` — 单个路径状态的内存大小
- **返回值**: `int` — 推荐的并发状态数量
- **算法**:
  - 基础值 = 多处理器数量 x 每处理器最大线程数 x 16（回退值 65536 x 16）
  - 支持通过环境变量 `CYCLES_CONCURRENT_STATES_FACTOR` 调整倍率
  - 最小值限制为 1024

### `HIPDeviceQueue::num_concurrent_busy_states()`
- **返回值**: `int` — 活跃忙碌状态数量
- **算法**: 多处理器数量 x 每处理器最大线程数 x 4（回退值 65536）

### `HIPDeviceQueue::init_execution()`
- **功能**: 任务执行前的初始化，同步所有纹理信息和内存拷贝
- 调用 `hip_device_->load_texture_info()` 上传纹理元数据
- 调用 `hipDeviceSynchronize()` 全局同步
- 调用 `debug_init_execution()` 启动调试追踪

### `HIPDeviceQueue::enqueue()`
- **参数**: `DeviceKernel kernel`, `int work_size`, `DeviceKernelArguments &args`
- **返回值**: `bool` — 是否成功
- **功能**: 内核分派的核心流程：
  1. 检查设备错误状态
  2. 更新纹理信息（内存可能被移到主机）
  3. 从 `HIPDeviceKernels` 获取内核的最优线程块大小
  4. 计算网格大小：`num_blocks = ceil(work_size / num_threads_per_block)`
  5. 对路径排序/压缩类内核分配共享内存：`(num_threads_per_block + 1) * sizeof(int)`
  6. 调用 `hipModuleLaunchKernel()` 启动内核
- **共享内存分配的特殊内核**: `INTEGRATOR_QUEUED_PATHS_ARRAY`、`INTEGRATOR_ACTIVE_PATHS_ARRAY`、`INTEGRATOR_SORTED_PATHS_ARRAY`、`INTEGRATOR_COMPACT_PATHS_ARRAY` 等路径数组操作内核需要共享内存用于并行索引计算

### `HIPDeviceQueue::synchronize()`
- 调用 `hipStreamSynchronize(hip_stream_)` 等待流中所有操作完成
- 检查设备错误状态

### `HIPDeviceQueue::zero_to_device()`
- 按需分配设备内存（如尚未分配）
- 调用 `hipMemsetD8Async()` 在流上异步清零

### `HIPDeviceQueue::copy_to_device()`
- 按需分配设备内存
- 调用 `hipMemcpyHtoDAsync()` 异步拷贝主机数据到设备

### `HIPDeviceQueue::copy_from_device()`
- 调用 `hipMemcpyDtoHAsync()` 异步拷贝设备数据到主机

### `HIPDeviceQueue::assert_success()`
- **功能**: HIP API 返回值检查辅助函数
- 错误时调用 `hip_device_->set_error()`，错误信息包含操作名称和当前活跃内核列表

### `HIPDeviceQueue::graphics_interop_create()`
- 创建并返回 `HIPDeviceGraphicsInterop` 实例

## 依赖关系
- **内部头文件**:
  - `device/memory.h` — `device_memory` 内存描述类
  - `device/queue.h` — `DeviceQueue` 基类
  - `device/hip/util.h` — `HIPContextScope`、错误检查宏
  - `device/hip/device_impl.h` — `HIPDevice` 类
  - `device/hip/graphics_interop.h` — `HIPDeviceGraphicsInterop` 图形互操作
  - `device/hip/kernel.h` — `HIPDeviceKernel`、`HIPDeviceKernels` 内核缓存
- **被引用**: `src/device/hip/device_impl.h`（`HIPDevice` 包含 `queue.h`），`src/device/hip/device_impl.cpp`（`reserve_local_memory()` 中使用 `HIPDeviceQueue`）

## 实现细节 / 关键算法

- **非阻塞流**: 使用 `hipStreamNonBlocking` 标志创建流，使其不会被 HIP 默认流的隐式同步阻塞，允许多个队列并行操作。
- **按需分配**: `zero_to_device()` 和 `copy_to_device()` 在设备指针为空时自动调用 `mem_alloc()` 分配内存，简化上层调用逻辑。
- **共享内存用途**: 路径数组操作内核使用共享内存实现并行前缀扫描（参见 `parall_active_index.h`），需要 `(线程数+1) * sizeof(int)` 字节的共享内存。
- **纹理信息同步**: 每次内核分派前检查并上传纹理元数据，处理内存因容量不足被移到主机的情况。
- **并发状态数估算**: 16 倍于硬件最大并发线程数的设计允许波前路径追踪器维护足够多的飞行中路径状态，以隐藏内存延迟。

## 关联文件
- `src/device/hip/device_impl.h` / `device_impl.cpp` — `HIPDevice::gpu_queue_create()` 创建本队列
- `src/device/hip/kernel.h` / `kernel.cpp` — 提供内核函数句柄和启动参数
- `src/device/hip/graphics_interop.h` / `graphics_interop.cpp` — 由本队列创建的图形互操作对象
- `src/device/queue.h` — `DeviceQueue` 基类定义
- `src/integrator/` — 波前路径追踪积分器通过队列提交内核
