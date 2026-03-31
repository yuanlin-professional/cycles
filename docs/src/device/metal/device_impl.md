# device_impl.h / device_impl.mm - Metal 设备的核心实现

## 概述

本文件是 Cycles Metal 后端的核心实现，定义了 `MetalDevice` 类，继承自通用 `Device` 基类。它负责管理 Metal 设备的完整生命周期，包括设备初始化、内核编译与加载、GPU 内存分配与释放、纹理管理、层次包围体（BVH）构建、命令队列创建，以及与 MetalRT 硬件光线追踪的集成。

## 类与结构体

### MetalDevice
- **继承**: `Device`
- **功能**: Metal GPU 渲染设备的完整实现，管理所有 GPU 资源和渲染管线
- **关键成员**:
  - `mtlDevice` (`id<MTLDevice>`) — 底层 Metal 设备句柄
  - `mtlLibrary[PSO_NUM]` (`id<MTLLibrary>`) — 编译后的 Metal 着色器库数组，按管线类型索引
  - `mtlComputeCommandQueue` (`id<MTLCommandQueue>`) — 路径追踪计算命令队列
  - `mtlGeneralCommandQueue` (`id<MTLCommandQueue>`) — 通用命令队列
  - `launch_params_buffer` (`id<MTLBuffer>`) — 内核启动参数缓冲区
  - `launch_params` (`KernelParamsMetal*`) — 直接映射到统一内存的内核参数指针
  - `use_metalrt` (`bool`) — 是否启用 MetalRT 硬件光线追踪
  - `accel_struct` (`id<MTLAccelerationStructure>`) — 顶层加速结构
  - `blas_array` / `unique_blas_array` — 底层加速结构数组
  - `kernel_features` (`uint`) — 当前场景所需的内核特性位掩码
  - `metal_mem_map` (`MetalMemMap`) — 设备内存分配映射表
  - `texture_info` (`device_vector<TextureInfo>`) — 纹理信息数组
  - `texture_bindings` (`id<MTLBuffer>`) — 无绑定纹理的 GPU 地址缓冲区
  - `kernel_specialization_level` (`MetalPipelineType`) — 内核特化级别
  - `max_threads_per_threadgroup` — 每线程组最大线程数（默认 512）
  - `device_id` — 设备唯一 ID，用于异步编译请求的生命周期管理
- **关键方法**:
  - `load_kernels(kernel_features)` — 触发内核编译（异步），生成 Metal 着色器库
  - `compile_and_load(device_id, pso_type)` — 静态方法，执行 MSL->AIR 前端编译，然后触发 PSO 后端编译
  - `make_source(pso_type, kernel_features)` — 生成完整的 Metal 内核源码
  - `preprocess_source(pso_type, kernel_features, source)` — 预处理源码，注入宏定义和 Metal function constants 特化
  - `build_bvh(bvh, progress, refit)` — 构建 BVH 加速结构
  - `optimize_for_scene(scene)` — 根据场景特征触发内核特化编译
  - `gpu_queue_create()` — 创建 `MetalDeviceQueue` 实例
  - `generic_alloc(mem)` / `generic_free(mem)` — 底层内存分配/释放
  - `tex_alloc(mem)` / `tex_free(mem)` — 纹理分配/释放
  - `const_copy_to(name, host, size)` — 将常量数据复制到内核参数缓冲区
  - `flush_delayed_free_list()` — 刷新延迟释放列表
  - `is_ready(status)` — 查询设备就绪状态（内核编译进度）

### MetalDevice::MetalMem（内部结构体）
- **功能**: 封装单个 Metal 内存分配的元数据
- **关键成员**:
  - `mem` (`device_memory*`) — 关联的设备内存对象
  - `pointer_index` (`int`) — 在启动参数中的指针索引
  - `mtlBuffer` (`id<MTLBuffer>`) — Metal 缓冲区
  - `mtlTexture` (`id<MTLTexture>`) — Metal 纹理（用于 2D/3D 纹理）
  - `offset`, `size` — 分配的偏移和大小
  - `hostPtr` (`void*`) — 统一内存的主机端指针

## 核心函数

- `get_device_by_ID(ID, lock)` — 线程安全地通过 ID 查找活跃的 MetalDevice 实例
- `is_device_cancelled(ID)` — 检查指定 ID 的设备是否已被取消
- `refresh_source_and_kernels_md5(pso_type)` — 刷新源码和 MD5 校验值，用于检测内核是否需要重新编译

## 依赖关系

- **内部头文件**:
  - `bvh/bvh.h` — BVH 基类
  - `device/device.h` — 通用 Device 基类
  - `device/metal/bvh.h` — BVHMetal 加速结构
  - `device/metal/device.h` — Metal 设备工厂接口
  - `device/metal/kernel.h` — Metal 内核管线
  - `device/metal/queue.h` — Metal 命令队列
  - `device/metal/util.h` — Metal 工具函数
  - `scene/scene.h` — 场景管理
  - `util/debug.h`, `util/md5.h`, `util/path.h`, `util/time.h`
  - `kernel/device/metal/globals.h` — Metal 内核全局参数定义
- **被引用**:
  - `device/metal/device.mm` — 创建 MetalDevice 实例
  - `device/metal/graphics_interop.h` — 引用 MetalDevice 类型
  - `device/metal/graphics_interop.mm` — 使用 MetalDevice
  - `device/metal/kernel.mm` — 访问设备属性进行内核编译
  - `device/metal/queue.mm` — MetalDeviceQueue 使用 MetalDevice
  - `device/metal/util.mm` — 工具函数实现引用设备实现

## 实现细节 / 关键算法

1. **统一内存架构（UMA）**: Apple Silicon 使用统一内存，因此 `generic_copy_to()` 和 `mem_copy_from()` 实际上是空操作（no-op），数据通过共享内存直接访问。`generic_alloc()` 使用 `MTLResourceStorageModeShared` 分配缓冲区，并将 `host_pointer` 直接指向 Metal 缓冲区的内容指针。

2. **内核编译管线**: 采用三级 PSO（Pipeline State Object）策略：
   - `PSO_GENERIC` — 通用内核，支持所有场景特性，编译慢但只需编译一次
   - `PSO_SPECIALIZED_INTERSECT` — 特化的相交内核，利用 Metal function constants 替换 KernelData 变量，编译快且性能更优
   - `PSO_SPECIALIZED_SHADE` — 特化的着色内核，额外短路未使用的 SVM 节点，性能最优但编译最慢

3. **异步编译**: `load_kernels()` 通过 GCD (`dispatch_async`) 异步执行前端编译（MSL->AIR），完成后触发后端 PSO 编译。`is_ready()` 方法报告编译进度，确保 UI 可以显示加载状态。

4. **设备生命周期安全**: 使用 `existing_devices_mutex` + `active_device_ids` 映射表确保异步编译请求在设备销毁后能被安全取消，避免访问已释放的对象。

5. **内存管理**: `device_pointer` 被编码为 `MetalMem*` 指针，支持资源重定位和设备指针重算。延迟释放列表 (`delayed_free_list`) 确保 GPU 正在使用的资源不会被立即释放。

6. **纹理管理**: 支持 2D/3D Metal 纹理和 1D 缓冲区纹理两种模式。无绑定纹理通过 `texture_bindings` 缓冲区存储 GPU 地址/资源 ID，在内核启动时绑定。

## 关联文件

- `src/device/metal/device.h` / `device.mm` — 工厂接口和设备枚举
- `src/device/metal/kernel.h` / `kernel.mm` — 内核管线编译
- `src/device/metal/queue.h` / `queue.mm` — 命令队列实现
- `src/device/metal/bvh.h` / `bvh.mm` — BVH 加速结构构建
- `src/device/metal/util.h` / `util.mm` — Metal 工具函数
