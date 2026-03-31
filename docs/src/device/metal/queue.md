# queue.h / queue.mm - Metal 设备命令队列与内核调度

## 概述

本文件实现了 Cycles 渲染器 Metal 后端的命令队列系统。`MetalDeviceQueue` 类封装了 Metal 命令缓冲区和编码器的管理，负责将渲染内核调度到 GPU 执行。它处理内核参数编码、资源绑定、MetalRT 加速结构关联、GPU 同步、内存传输，并提供性能分析和 GPU 追踪捕获功能。

## 类与结构体

### MetalDeviceQueue
- **继承**: `DeviceQueue`
- **功能**: Metal GPU 命令队列的完整实现，管理从内核提交到完成同步的全部流程
- **关键成员**:
  - `metal_device_` (`MetalDevice*`) — 关联的 Metal 设备
  - `mtlDevice_` (`id<MTLDevice>`) — Metal 设备句柄
  - `mtlCommandQueue_` (`id<MTLCommandQueue>`) — Metal 命令队列
  - `mtlCommandBuffer_` (`id<MTLCommandBuffer>`) — 当前活跃的命令缓冲区
  - `mtlComputeEncoder_` (`id<MTLComputeCommandEncoder>`) — 当前活跃的计算命令编码器
  - `mtlBlitEncoder_` (`id<MTLBlitCommandEncoder>`) — 当前活跃的位块传输编码器
  - `shared_event_` (`id<MTLSharedEvent>`) — 用于 CPU-GPU 同步的共享事件
  - `shared_event_listener_` (`MTLSharedEventListener*`) — 共享事件监听器
  - `wait_semaphore_` (`dispatch_semaphore_t`) — 同步等待信号量
  - `active_pipelines_[DEVICE_KERNEL_NUM]` (`MetalDispatchPipeline`) — 每个内核的活跃管线数组
  - `command_buffer_desc_` (`MTLCommandBufferDescriptor*`) — 命令缓冲区描述符（启用增强错误报告）
  - `stats_` (`Stats&`) — 统计数据引用
- **关键方法**:
  - `enqueue(kernel, work_size, args)` — 核心调度方法，编码并提交内核到 GPU
  - `synchronize()` — 等待所有已提交命令完成
  - `init_execution()` — 初始化执行环境，填充 BLAS 数组和纹理绑定
  - `zero_to_device(mem)` — 在 GPU 端将内存清零
  - `copy_to_device(mem)` — 复制数据到设备端（UMA 下按需分配即可）
  - `copy_from_device(mem)` — 从设备复制数据（UMA 下为空操作）
  - `num_concurrent_states(state_size)` — 计算最优并发状态数
  - `num_concurrent_busy_states(state_size)` — 返回 busy 状态数（为总状态数的 1/4）
  - `num_sort_partitions(max_num_paths, max_scene_shaders)` — 计算排序分区数
  - `graphics_interop_create()` — 创建 `MetalDeviceGraphicsInterop` 实例
  - `native_queue()` — 返回底层 `MTLCommandQueue` 指针

### MetalDeviceQueue::TimingData（内部结构体）
- **功能**: 记录单次内核调度的性能数据
- **关键成员**: `kernel`, `work_size`, `timing_id`

### MetalDeviceQueue::TimingStats（内部结构体）
- **功能**: 累计每种内核的性能统计
- **关键成员**: `total_time`, `total_work_size`, `num_dispatches`

## 核心函数

- `get_compute_encoder(kernel)` — 获取或创建计算命令编码器，根据内核类型选择并发/串行调度模式
- `get_blit_encoder()` — 获取或创建位块传输编码器
- `close_compute_encoder()` / `close_blit_encoder()` — 结束并释放编码器
- `prepare_resources(kernel)` — 声明所有 Metal 资源的使用模式（读/写/采样）
- `setup_capture()` — 根据环境变量配置 GPU 追踪捕获
- `update_capture(kernel)` — 在每次调度时更新捕获状态
- `begin_capture()` / `end_capture()` — 开始/结束 GPU 追踪捕获
- `flush_timing_stats()` — 从 Counter Sample Buffer 中解析性能数据

## 依赖关系

- **内部头文件**:
  - `device/kernel.h` — DeviceKernel 枚举
  - `device/memory.h` — 设备内存管理
  - `device/queue.h` — DeviceQueue 基类
  - `device/metal/util.h` — Metal 工具函数（`metal_gpuAddress`、`metal_gpuResourceID`）
  - `device/metal/device_impl.h` — MetalDevice 类型
  - `device/metal/graphics_interop.h` — MetalDeviceGraphicsInterop
  - `device/metal/kernel.h` — MetalDispatchPipeline 和 MetalDeviceKernels
  - `kernel/device/metal/globals.h` — KernelParamsMetal 定义
- **被引用**:
  - `device/metal/device_impl.h` — MetalDevice 引用 MetalDeviceQueue 类型

## 实现细节 / 关键算法

1. **内核调度流程** (`enqueue`):
   - 获取/复用计算命令编码器
   - 更新管线（`active_pipeline.update()`），必要时热切换到更优的特化管线
   - 编码动态参数到字节缓冲区（slot 0），处理指针类型参数的资源声明
   - 编码启动参数缓冲区（slot 1）和辅助参数（slot 2，包含纹理绑定、加速结构、相交函数表等）
   - 设置共享内存大小（用于原子排序等内核）
   - 调用 `dispatchThreads:threadsPerThreadgroup:` 提交调度
   - 添加 `completedHandler` 回调以捕获命令缓冲区错误

2. **并发与串行调度**: 路径追踪积分器内核（`< DEVICE_KERNEL_INTEGRATOR_NUM`）使用并发调度（`MTLDispatchTypeConcurrent`），其他内核使用串行调度，以平衡 GPU 利用率。

3. **同步机制**: 使用 `MTLSharedEvent` + `MTLSharedEventListener` 实现 CPU-GPU 同步。`synchronize()` 注册事件监听器并等待信号量，避免了 `waitUntilCompleted` 的忙等开销。

4. **UMA 内存优化**: 由于 Apple Silicon 统一内存架构，`copy_to_device()` 仅在首次使用时按需分配内存，`copy_from_device()` 为空操作。`zero_to_device()` 优先使用 Blit 编码器的 `fillBuffer` 命令进行 GPU 端清零。

5. **并发状态数计算**: 基础值为 4194304，对非 M1 架构在内存充足时加倍（需要至少保留系统 RAM 的 1/8 或 1GB 作为缓冲）。busy 状态数为总数的 1/4。

6. **性能分析**: 通过 `CYCLES_METAL_PROFILING` 环境变量启用，使用 `MTLCounterSampleBuffer` 收集每个编码器的 GPU 时间戳，在队列析构时输出详细的每内核耗时统计表。

7. **GPU 追踪捕获**: 通过 `CYCLES_DEBUG_METAL_CAPTURE_KERNEL` 和 `CYCLES_DEBUG_METAL_CAPTURE_SAMPLES` 环境变量控制，支持捕获单次调度或采样块到 `.gputrace` 文件。

## 关联文件

- `src/device/metal/device_impl.h` / `device_impl.mm` — MetalDevice 创建本队列
- `src/device/metal/kernel.h` / `kernel.mm` — 提供 MetalDispatchPipeline 和管线缓存
- `src/device/metal/graphics_interop.h` / `graphics_interop.mm` — 由本队列创建的图形互操作对象
- `src/device/metal/util.h` / `util.mm` — GPU 地址和资源 ID 辅助函数
