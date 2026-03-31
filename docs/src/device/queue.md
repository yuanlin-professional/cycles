# queue.h / queue.cpp - 设备命令队列抽象与内核调度

## 概述

本文件定义了 Cycles 渲染器的设备命令队列抽象层，包括内核参数容器 `DeviceKernelArguments` 和命令队列基类 `DeviceQueue`。`DeviceQueue` 提供了内核入队执行、同步等待、内存操作和图形互操作的统一接口，是波前（Wavefront）路径追踪架构中 GPU 内核调度的核心组件。实现文件包含调试日志和性能统计功能。

## 类与结构体

### DeviceKernelArguments
- **继承**: 无
- **功能**: 类型安全的设备内核参数容器，封装内核启动时所需的全部参数。通过模板方法自动推导参数类型。
- **关键成员**:
  - `types[MAX_ARGS]` — 各参数的类型标识数组
  - `values[MAX_ARGS]` — 各参数的值指针数组
  - `sizes[MAX_ARGS]` — 各参数的字节大小数组
  - `count` — 当前参数个数
  - `MAX_ARGS = 18` — 最大支持参数数量
- **关键方法**:
  - `add(const device_ptr *value)` — 添加设备指针参数（类型 `POINTER`）
  - `add(const int32_t *value)` — 添加 32 位整数参数（类型 `INT32`）
  - `add(const float *value)` — 添加 32 位浮点参数（类型 `FLOAT32`）
  - `add(const KernelFilmConvert *value)` — 添加胶片转换参数结构体（类型 `KERNEL_FILM_CONVERT`）
  - 变参模板构造函数 — 支持一次性传入多个参数

### DeviceKernelArguments::Type
- `POINTER` — 设备指针
- `INT32` — 32 位整数
- `FLOAT32` — 32 位浮点数
- `KERNEL_FILM_CONVERT` — 胶片转换参数结构体
- `HIPRT_GLOBAL_STACK` — HIP RT 全局堆栈

### DeviceQueue
- **继承**: 无（抽象基类）
- **功能**: 设备命令队列的统一接口，封装内核执行调度、同步、内存操作和图形互操作。各 GPU 后端（CUDA、HIP、Metal、oneAPI、OptiX）继承此类提供具体实现。
- **关键成员**:
  - `device` — 队列所属设备指针（`Device*`）
  - `last_kernels_enqueued_` — 上次同步以来已入队的内核掩码（`DeviceKernelMask`）
  - `last_sync_time_` — 上次同步时间戳
  - `stats_kernel_time_` — 按内核组合分类的累积执行时间（`map<DeviceKernelMask, double>`）
  - `is_per_kernel_performance_` — 是否启用逐内核性能追踪模式
- **关键方法**:
  - `num_concurrent_states(state_size)` — 纯虚方法，返回积分器可并发处理的状态数量（基于核心数或可用内存）
  - `num_concurrent_busy_states(state_size)` — 纯虚方法，返回保持设备满负载所需的最少活跃状态数量
  - `num_sort_partitions(max_num_paths, max_scene_shaders)` — 返回着色器排序分区数量，当着色器数量小于 300 时按 `max_num_paths / 65536` 分区以提升内存局部性
  - `supports_local_atomic_sort()` — 是否支持本地原子排序内核
  - `init_execution()` — 纯虚方法，初始化内核执行（加载数据到全局或路径状态）
  - `enqueue(kernel, work_size, args)` — 纯虚方法，入队内核执行
  - `synchronize()` — 纯虚方法，等待所有已入队内核完成
  - `zero_to_device(mem)` / `copy_to_device(mem)` / `copy_from_device(mem)` — 命令队列内的内存操作（保证执行顺序）
  - `graphics_interop_create()` — 创建图形互操作上下文（默认实现为不支持）
  - `native_queue()` — 返回底层原生队列句柄

## 核心函数

- `DeviceQueue::DeviceQueue(device)` — 构造函数，通过环境变量 `CYCLES_DEBUG_PER_KERNEL_PERFORMANCE` 控制逐内核性能模式。
- `DeviceQueue::~DeviceQueue()` — 析构函数，在 TRACE 日志级别下打印按执行时间降序排列的内核组合性能统计。
- `debug_init_execution()` — 记录同步起始时间，重置内核掩码。
- `debug_enqueue_begin(kernel, work_size)` — 记录内核入队事件到日志，将内核加入当前掩码。
- `debug_enqueue_end()` — 在逐内核性能模式下，每次入队后立即强制同步以获取精确计时。
- `debug_synchronize()` — 计算自上次同步以来的耗时，累加到对应内核组合的统计条目中。
- `debug_active_kernels()` — 返回当前已入队内核的字符串表示。

## 依赖关系

- **内部头文件**: `device/kernel.h`、`device/graphics_interop.h`、`util/log.h`、`util/map.h`、`util/string.h`、`util/unique_ptr.h`
- **实现文件额外依赖**: `util/algorithm.h`、`util/time.h`
- **外部库**: 无
- **被引用**:
  - `src/device/device.cpp` — `Device::gpu_queue_create()` 使用 `DeviceQueue`
  - `src/device/multi/device.cpp` — 多设备队列管理
  - `src/device/dummy/device.cpp` — 占位设备队列
  - `src/integrator/path_trace_work_gpu.h` — GPU 波前路径追踪的核心调度使用者
  - `src/integrator/shader_eval.cpp` — 着色器求值内核调度
  - `src/integrator/work_tile_scheduler.cpp` — 工作块调度器
  - `src/integrator/pass_accessor_gpu.cpp` — GPU 通道访问器
  - `src/integrator/denoiser_gpu.cpp` — GPU 降噪器内核调度
  - `src/integrator/denoiser_oidn.cpp` — OIDN 降噪器

## 实现细节 / 关键算法

- **性能统计架构**: `DeviceQueue` 使用 `DeviceKernelMask` 作为键的 `map` 来分组统计内核执行时间。同一次 `init_execution()` 到 `synchronize()` 之间入队的所有内核被视为一个组合，共享一条时间记录。析构时将所有组合按耗时降序排列打印。
- **逐内核性能模式**: 当环境变量 `CYCLES_DEBUG_PER_KERNEL_PERFORMANCE` 被设置时，`debug_enqueue_end()` 会在每次内核入队后立即调用 `synchronize()`，使每个内核的执行时间可以被精确测量，但会显著降低整体性能。
- **排序分区启发式**: `num_sort_partitions()` 在着色器数量超过 300 时禁用分区（返回 1），因为大量着色器会降低分区的内存局部性收益。分区数量为 `max_num_paths / 65536`，至少为 1。
- **参数类型安全**: `DeviceKernelArguments` 通过多个 `add()` 重载和模板变参构造函数确保每种参数类型被正确识别，避免参数传递中的类型错误。`MAX_ARGS = 18` 覆盖了当前所有内核的最大参数数量需求。

## 关联文件

- `src/device/kernel.h` — 提供 `DeviceKernel` 枚举和 `DeviceKernelMask` 类型
- `src/device/graphics_interop.h` — 图形互操作接口
- `src/device/device.h` — `Device::gpu_queue_create()` 返回 `DeviceQueue`
- `src/integrator/path_trace_work_gpu.cpp` — 波前路径追踪中通过队列调度全部内核
- `src/device/cuda/queue.h` — CUDA 后端队列实现
- `src/device/hip/queue.h` — HIP 后端队列实现
- `src/device/metal/queue.h` — Metal 后端队列实现
- `src/device/oneapi/queue.h` — oneAPI 后端队列实现
- `src/device/optix/queue.h` — OptiX 后端队列实现
