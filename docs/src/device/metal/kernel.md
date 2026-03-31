# kernel.h / kernel.mm - Metal 内核管线状态对象（PSO）管理与着色器缓存

## 概述

本文件实现了 Cycles 渲染器 Metal 后端的内核编译、管线状态对象（PSO）管理和着色器缓存系统。它定义了三种管线特化级别（通用/相交特化/着色特化），通过 Metal function constants 实现内核特化，使用多线程编译池和二进制归档（Binary Archive）缓存来优化编译性能。这是 Metal 后端性能优化的核心模块。

## 类与结构体

### MetalKernelPipeline
- **功能**: 可在多个 `MetalDeviceQueue` 实例之间共享的管线对象，封装从 Metal 函数到计算管线状态的完整编译产物
- **关键成员**:
  - `pipeline_id` (`int`) — 管线唯一标识符
  - `originating_device_id` (`int`) — 创建该管线的设备 ID
  - `mtlLibrary` (`id<MTLLibrary>`) — 编译后的 Metal 着色器库
  - `pso_type` (`MetalPipelineType`) — 管线特化类型
  - `kernels_md5` (`string`) — 内核源码的 MD5 校验值，用于缓存匹配
  - `kernel_data_` (`KernelData`) — 用于特化的内核数据快照
  - `use_metalrt` (`bool`) — 是否使用 MetalRT
  - `device_kernel` (`DeviceKernel`) — 对应的设备内核枚举
  - `function` (`id<MTLFunction>`) — Metal 计算函数
  - `pipeline` (`id<MTLComputePipelineState>`) — 编译后的计算管线状态
  - `table_functions[METALRT_TABLE_NUM]` — MetalRT 相交函数表数组
- **关键方法**:
  - `compile()` — 执行完整的管线编译流程，包括函数获取、链接函数设置、PSO 创建和二进制归档
  - `should_use_binary_archive()` — 判断是否应使用二进制归档缓存（需 macOS 15.4+）
  - `make_intersection_function(function_name)` — 创建 MetalRT 相交函数

### MetalDispatchPipeline
- **功能**: 绑定到单个 `MetalDeviceQueue` 的活跃管线实例，持有设备特定的相交函数表
- **关键成员**:
  - `pipeline_id` (`int`) — 当前使用的管线 ID
  - `pipeline` (`id<MTLComputePipelineState>`) — 计算管线状态
  - `intersection_func_table[METALRT_TABLE_NUM]` — MetalRT 相交函数表实例
- **关键方法**:
  - `update(metal_device, kernel)` — 检查并切换到最优管线版本
  - `free_intersection_function_tables()` — 释放相交函数表资源

### ShaderCache（内部类，定义在 kernel.mm）
- **功能**: 全局着色器编译缓存，管理编译线程池和管线对象集合
- **关键成员**:
  - `pipelines[DEVICE_KERNEL_NUM]` (`PipelineCollection`) — 每个内核类型的管线集合
  - `request_queue` (`deque<MetalKernelPipeline>`) — 待编译请求队列
  - `compile_threads` — 编译线程池
  - `occupancy_tuning[DEVICE_KERNEL_NUM]` — 每架构每内核的占用率调优参数
- **关键方法**:
  - `load_kernel(kernel, device, pso_type)` — 提交内核编译请求
  - `get_best_pipeline(kernel, device)` — 获取当前最优管线（阻塞等待直到可用）
  - `should_load_kernel(kernel, device, pso_type)` — 检查内核是否需要编译
  - `compile_thread_func()` — 编译线程主循环

### MetalPipelineType（枚举）
- `PSO_GENERIC` — 通用管线，支持所有特性，编译慢但只需一次
- `PSO_SPECIALIZED_INTERSECT` — 特化相交管线，使用 function constants 替换 KernelData 变量
- `PSO_SPECIALIZED_SHADE` — 特化着色管线，额外短路未使用的 SVM 节点

### METALRT_TABLE_*（枚举）
- 定义 8 种 MetalRT 相交函数表类型：默认、阴影、全阴影、体积、局部、局部运动模糊、局部单次命中、局部单次命中运动模糊

## 核心函数（MetalDeviceKernels 命名空间）

- `load(device, pso_type)` — 为设备加载指定类型的全部内核
- `get_best_pipeline(device, kernel)` — 获取设备上指定内核的最优管线
- `wait_for_all()` — 等待所有着色器缓存完成编译
- `num_incomplete_specialization_requests()` — 返回未完成的特化编译请求数
- `should_load_kernels(device, pso_type)` — 检查是否有内核需要加载
- `is_benchmark_warmup()` — 检测是否处于基准测试预热阶段
- `static_deinitialize()` — 释放所有静态资源
- `kernel_type_as_string(pso_type)` — 将管线类型转换为可读字符串

## 依赖关系

- **内部头文件**:
  - `device/kernel.h` — DeviceKernel 枚举定义
  - `device/metal/device_impl.h` — MetalDevice 类型
  - `kernel/device/metal/function_constants.h` — Metal function constants 索引定义
  - `util/debug.h`, `util/md5.h`, `util/path.h`, `util/tbb.h`, `util/time.h`
- **被引用**:
  - `device/metal/device_impl.h` — 引用 MetalPipelineType 和 MetalDeviceKernels
  - `device/metal/queue.mm` — MetalDeviceQueue 使用 MetalDispatchPipeline
  - `device/metal/util.h` — 引用 MetalPipelineType

## 实现细节 / 关键算法

1. **三级特化策略**: 渲染开始时先加载 `PSO_GENERIC` 通用内核（快速启动渲染），然后在后台异步编译特化内核。`MetalDispatchPipeline::update()` 在每次调度时检查是否有更优的特化管线可用，并无缝热切换。

2. **Metal Function Constants**: 特化管线通过 `GetConstantValues()` 函数将 `KernelData` 结构体的成员值注入为编译时常量，使 Metal 编译器能够优化掉未使用的代码路径。源码中的 `kernel_data.parent.name` 访问被替换为 `kernel_data_parent_name` function constant 标识符。

3. **二进制归档缓存**: 在 macOS 15.4+ 上，编译后的 PSO 通过 `MTLBinaryArchive` 序列化到磁盘缓存。下次渲染时可直接加载，避免重复编译。缓存文件路径基于设备名、内核名、PSO 类型和内容 MD5 生成。损坏的归档会被自动检测和重建。

4. **占用率调优**: `ShaderCache` 为不同 Apple GPU 架构（M1/M2/M2 Big/M3）预设了每个内核的最优线程组大小和线程块大小。M3 架构依赖 Dynamic Caching 机制，统一使用 64 线程。

5. **编译线程池**: 编译线程数默认为 2，在 macOS 13.3+ 上通过 `maximumConcurrentCompilationTaskCount` 查询上限并减 1（避免与实时 GPU 模块竞争）。每种 PSO 类型最多缓存 3 个同类管线，超出时淘汰最旧的。

6. **相交函数表**: 对启用 MetalRT 的相交内核，编译时会创建 8 种相交函数表，分别处理三角形、曲线和点云的不同光线类型（默认/阴影/体积/局部）。这些函数通过 `MTLLinkedFunctions` 链接到计算管线。

## 关联文件

- `src/device/metal/device_impl.h` / `device_impl.mm` — 触发内核编译和特化
- `src/device/metal/queue.h` / `queue.mm` — 使用编译后的管线进行内核调度
- `src/kernel/device/metal/function_constants.h` — function constant 索引定义
- `src/kernel/data_template.h` — KernelData 结构体成员宏定义
