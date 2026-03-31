# device.h / device.cpp - 设备抽象层核心定义与工厂实现

## 概述

本文件是 Cycles 渲染器设备抽象层的核心，定义了设备类型枚举、设备信息描述结构 `DeviceInfo`、设备基类 `Device` 以及 GPU 设备基类 `GPUDevice`。它提供了统一的设备创建工厂、设备枚举发现、BVH 构建、内存管理等核心接口，是所有具体后端（CPU、CUDA、OptiX、HIP、Metal、oneAPI）的公共抽象基础。

## 类与结构体

### DeviceInfo
- **继承**: 无
- **功能**: 描述单个计算设备的静态信息和能力特征，用于设备选择和多设备组合。
- **关键成员**:
  - `type` — 设备类型（`DeviceType`），默认 `DEVICE_CPU`
  - `description` — 设备的人类可读描述字符串
  - `id` — 设备唯一标识符，用于用户偏好持久化
  - `num` — 设备编号
  - `display_device` — GPU 是否被用作显示设备
  - `has_nanovdb` — 是否支持 NanoVDB 体积
  - `has_mnee` — 是否支持 MNEE（Manifold Next Event Estimation）
  - `has_osl` — 是否支持 Open Shading Language
  - `has_guiding` — 是否支持路径追踪引导
  - `has_gpu_queue` — 是否支持 GPU 队列
  - `use_hardware_raytracing` — 是否使用硬件加速光线追踪
  - `kernel_optimization_level` — 内核优化等级（Metal 专用）
  - `denoisers` — 支持的降噪器类型位掩码
  - `multi_devices` — 多设备组合时包含的子设备列表
- **关键方法**:
  - `operator==` / `operator!=` — 基于 `id` 和硬件光追配置的相等比较

### Device
- **继承**: 无（抽象基类）
- **功能**: 所有计算设备的统一抽象接口，定义了内存管理、内核加载、BVH 构建、GPU 队列创建等核心操作。
- **关键成员**:
  - `info` — 设备信息（`DeviceInfo`）
  - `stats` — 内存统计引用（`Stats&`）
  - `profiler` — 性能分析器引用（`Profiler&`）
  - `headless` — 是否为无头模式
- **关键方法**:
  - `create(info, stats, profiler, headless)` — **静态工厂方法**，根据 `DeviceInfo` 创建对应后端的设备实例
  - `load_kernels(kernel_features)` — 加载/编译内核，必须在添加任务前调用
  - `build_bvh(bvh, progress, refit)` — 构建或重建 BVH 加速结构
  - `gpu_queue_create()` — 创建 GPU 命令队列
  - `get_cpu_kernels()` — 获取 CPU 内核函数集合（静态方法）
  - `mem_alloc()` / `mem_copy_to()` / `mem_free()` 等 — 设备内存管理纯虚方法
  - `const_copy_to()` — 常量内存拷贝
  - `available_devices(mask)` — 枚举系统中所有可用设备（静态方法）
  - `available_types()` — 返回当前编译支持的设备类型列表（静态方法）
  - `get_multi_device(subdevices, threads, background)` — 组合多个设备为一个多设备配置（静态方法）
  - `foreach_device(callback)` — 对每个实际渲染子设备执行回调
  - `should_use_graphics_interop()` — 检查是否应使用图形互操作
  - `get_guiding_device()` — 获取路径追踪引导设备句柄

### GPUDevice
- **继承**: `Device`
- **功能**: GPU 设备的通用基类，封装了纹理管理、主机映射内存、设备内存分配的通用逻辑。
- **关键成员**:
  - `texture_info` — 绑定纹理信息数组（`device_vector<TextureInfo>`）
  - `need_texture_info` — 纹理信息是否需要更新
  - `can_map_host` — 是否支持主机映射内存
  - `map_host_used` / `map_host_limit` — 主机映射内存使用量和限制
  - `device_mem_map` — 设备内存分配映射表
  - `device_mem_in_use` — 设备内存使用量追踪
- **关键方法**:
  - `load_texture_info()` — 加载纹理信息到设备
  - `init_host_memory()` — 初始化主机映射内存限制（基于系统 RAM）
  - `move_textures_to_host()` — 当设备内存不足时将纹理迁移到主机内存
  - `generic_alloc()` — 通用内存分配（优先设备内存，回退到主机映射内存）
  - `generic_free()` — 通用内存释放（支持共享内存引用计数）
  - `generic_copy_to()` — 通用的主机到设备内存拷贝

## 核心函数

- `Device::create()` — 核心工厂方法，通过 `switch` 根据设备类型分发到各后端的创建函数（`device_cpu_create`、`device_cuda_create` 等）。当创建失败时回退到 `device_dummy_create`。
- `Device::available_devices()` — 惰性初始化各后端设备列表，使用互斥锁和 `devices_initialized_mask` 位掩码确保每个后端只初始化一次。
- `Device::get_multi_device()` — 将多个子设备组合为一个 `DeviceInfo`，合并能力标志（取交集），生成复合 ID。对于后台渲染会减少 CPU 线程数以给 GPU 让路。
- `GPUDevice::generic_alloc()` — 分配策略：先检查设备内存是否充足（考虑 headroom），不足时调用 `move_textures_to_host()` 腾出空间，最终可回退到共享主机内存（`shared_alloc`）。
- `GPUDevice::move_textures_to_host()` — 按大小排序找到最大的纹理分配并迁移，优先迁移图像纹理。

## 枚举类型

### DeviceType
- `DEVICE_NONE`、`DEVICE_CPU`、`DEVICE_CUDA`、`DEVICE_MULTI`、`DEVICE_OPTIX`、`DEVICE_HIP`、`DEVICE_HIPRT`、`DEVICE_METAL`、`DEVICE_ONEAPI`、`DEVICE_DUMMY`

### DeviceTypeMask
- 每个设备类型对应的位掩码，用于 `available_devices()` 过滤

### KernelOptimizationLevel
- `KERNEL_OPTIMIZATION_LEVEL_OFF`、`KERNEL_OPTIMIZATION_LEVEL_INTERSECT`、`KERNEL_OPTIMIZATION_LEVEL_FULL` — 路径追踪内核的优化等级（Metal 专用）

### MetalRTSetting
- `METALRT_OFF`、`METALRT_ON`、`METALRT_AUTO` — Metal 硬件光线追踪设置

## 依赖关系

- **内部头文件**: `bvh/params.h`、`device/denoise.h`、`device/memory.h`、`device/queue.h`、`util/profiling.h`、`util/stats.h`、`util/string.h`、`util/texture.h`、`util/thread.h`、`util/types.h`、`util/unique_ptr.h`、`util/vector.h`
- **实现文件额外依赖**: `bvh/bvh2.h`、各后端的 `device/*/device.h`、`device/cpu/kernel.h`
- **外部库**: `hiprtew.h`（条件编译，HIP RT 支持）
- **被引用**: 被几乎所有场景管理模块、集成器模块、会话模块引用（共 50+ 个文件），是设备层的核心头文件

## 实现细节 / 关键算法

- **惰性设备发现**: `available_devices()` 使用静态成员 `devices_initialized_mask` 跟踪哪些后端已初始化。首次请求某类型设备时才调用对应的 `device_*_init()` 和 `device_*_info()` 函数，避免在未使用的平台上触发潜在的驱动崩溃。
- **主机内存限制策略**: `GPUDevice::init_host_memory()` 限制主机映射内存总量不超过系统 RAM 的一半或 `系统 RAM - 4GB`（取较小者），防止系统不稳定。
- **设备内存 Headroom**: `generic_alloc()` 为纹理保留 128MB headroom，为工作内存保留 32MB headroom，确保关键工作内存总有可用空间。
- **共享内存引用计数**: `generic_free()` 中对 `shared_pointer` 进行引用计数管理，多设备共享同一主机内存时只有最后一个释放者真正执行 `shared_free()`。
- **多设备 CPU 线程调整**: `get_multi_device()` 在后台渲染模式下，将 CPU 线程数减去 GPU 设备数量，避免 CPU 与 GPU 竞争资源。在交互式渲染中完全禁用 CPU 设备。

## 关联文件

- `src/device/cpu/device.cpp` — CPU 设备后端实现
- `src/device/cuda/device.cpp` — CUDA 设备后端实现
- `src/device/optix/device.cpp` — OptiX 设备后端实现
- `src/device/hip/device.cpp` — HIP 设备后端实现
- `src/device/metal/device.mm` — Metal 设备后端实现
- `src/device/oneapi/device.cpp` — oneAPI 设备后端实现
- `src/device/multi/device.cpp` — 多设备组合实现
- `src/device/dummy/device.cpp` — 占位/回退设备实现
- `src/device/memory.h` — 设备内存管理类
- `src/device/queue.h` — 设备命令队列类
- `src/device/denoise.h` — 降噪器参数定义
