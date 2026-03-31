# device_impl.h / device_impl.cpp - oneAPI 设备核心实现

## 概述

本文件定义并实现了 `OneapiDevice` 类，这是 Cycles 渲染器 oneAPI 后端的核心设备类。它继承自 `GPUDevice`，通过 Intel SYCL/Level-Zero 运行时提供完整的 GPU 计算能力，包括内存管理（USM 统一共享内存）、内核加载与执行、纹理管理、BVH 构建（可选 Embree GPU 硬件光线追踪）、图形互操作以及设备枚举与能力查询。整个文件受 `WITH_ONEAPI` 编译宏保护。

## 类与结构体

### `OneapiDevice`
- **继承**: `GPUDevice`（来自 `device/device.h`）
- **功能**: 封装基于 Intel oneAPI/SYCL 的 GPU 设备，为 Cycles 路径追踪渲染器提供设备抽象实现
- **关键成员**:
  - `device_queue_` (`SyclQueue*`) — 底层 SYCL 队列的不透明指针，实际类型为 `sycl::queue*`
  - `embree_device` (`RTCDevice`) — Embree GPU 光线追踪设备句柄（条件编译 `WITH_EMBREE_GPU`）
  - `embree_traversable` (`RTCTraversable` / `RTCScene`) — Embree BVH 遍历句柄
  - `const_mem_map_` (`ConstMemMap`) — 常量内存名称到设备向量的映射表
  - `kg_memory_` / `kg_memory_device_` (`void*`) — 内核全局数据（KernelGlobals）的主机端和设备端副本
  - `kg_memory_size_` (`size_t`) — 内核全局数据段大小
  - `max_memory_on_device_` (`size_t`) — 设备最大显存容量
  - `oneapi_error_string_` (`std::string`) — 运行时错误信息缓冲
  - `use_hardware_raytracing` (`bool`) — 是否启用硬件光线追踪
  - `kernel_features` (`unsigned int`) — 累积的内核特性掩码
  - `scene_max_shaders_` (`int`) — 当前场景最大着色器数量
  - `is_several_intel_dgpu_devices_detected` (`bool`) — 是否检测到多块 Intel 独立 GPU（用于启用兼容性变通方案）

### `OneAPIDeviceIteratorCallback`
- **类型**: 函数指针类型别名
- **签名**: `void (*)(const char *id, const char *name, const int num, bool hwrt_support, bool oidn_support, bool has_execution_optimization, void *user_ptr)`
- **功能**: 设备枚举回调，用于 `iterate_devices()` 方法

## 核心函数

### 构造与析构

#### `OneapiDevice()` (构造函数)
- 调用 `create_queue()` 创建 SYCL 有序队列
- 可选创建 Embree SYCL 设备用于硬件光线追踪
- 分配内核全局内存段（主机端 + 设备端），使用 `usm_aligned_alloc_host` 和 `usm_alloc_device`
- 通过 `get_memcapacity()` 获取设备总显存
- 支持通过 `CYCLES_ONEAPI_MEMORY_HEADROOM` 环境变量自定义显存预留量

#### `~OneapiDevice()` (析构函数)
- 释放 Embree 设备、纹理信息、内核全局内存、常量内存映射表和 SYCL 队列

### 内核管理

#### `load_kernels(requested_features)`
- 累积合并请求的内核特性到 `kernel_features` 掩码
- 执行测试内核以验证设备队列可用性
- 调用 `oneapi_load_kernels()` 编译/加载所需内核
- 调用 `reserve_private_memory()` 预分配内核执行所需的私有内存

#### `reserve_private_memory(kernel_features)`
- 选择最大的着色内核（按特性优先级：RAYTRACE > MNEE > SURFACE）
- 使用最小工作组大小（1）执行一次内核来触发运行时内存预留
- 记录预留前后的显存差值

#### `enqueue_kernel(kernel_context, kernel, global_size, local_size, args)`
- 委托给 `oneapi_enqueue_kernel()` 执行实际的内核派发

#### `get_adjusted_global_and_local_sizes(queue, kernel, global_size, local_size)`
- 根据内核类型选择最优工作组大小：
  - 交叉（intersect）内核: 128
  - 着色（shade）内核: 256（SIMD8 设备为 64）
  - Cryptomatte 内核: 512
  - 着色器求值内核: 256
  - 默认: 1024
- 将全局大小向上取整为工作组大小的整数倍（oneAPI 要求均匀工作组）

### 内存管理

#### USM 内存操作
- `usm_aligned_alloc_host(queue, size, alignment)` — 调用 `sycl::aligned_alloc_host`
- `usm_alloc_device(queue, size)` — 调用 `sycl::malloc_device`（调试模式下使用 `sycl::malloc_host`）
- `usm_free(queue, ptr)` — 调用 `sycl::free`
- `usm_memcpy(queue, dest, src, size)` — 智能拷贝：主机到主机使用 `memcpy`，涉及设备时使用 `sycl::queue::memcpy`；设备到主机传输会阻塞等待完成
- `usm_memset(queue, ptr, value, size)` — 调用 `sycl::queue::memset`

#### 设备内存
- `alloc_device(device_pointer, size)` — 分配设备内存后立即通过 `oneapi_zero_memory_on_device` 强制初始化，避免惰性分配导致的问题
- `free_device(device_pointer)` — 释放设备内存
- `get_device_memory_info(total, free)` — 查询设备总显存和可用显存

#### 共享内存
- `shared_alloc()` — 实际使用 USM 主机内存分配（`usm_aligned_alloc_host`），因为 Cycles 架构已明确区分主机/设备指针
- `shared_to_device_pointer()` — 在 USM 地址空间中直接返回同一指针

#### 主机内存优化
- `host_alloc()` — 调用基类分配后，使用 `sycl::ext::oneapi::experimental::prepare_for_device_copy` 将主机指针注册到 USM 以加速传输（仅 Level-Zero 后端，且仅在非多 dGPU 配置下启用）
- `host_free()` — 调用 `release_from_device_copy` 取消注册后释放

#### 通用内存接口
- `mem_alloc` / `mem_copy_to` / `mem_copy_from` / `mem_move_to_host` / `mem_zero` / `mem_free` — 按内存类型（`MEM_TEXTURE` / `MEM_GLOBAL` / 通用）分发到对应的 `tex_*`、`global_*` 或 `generic_*` 方法

#### 常量内存
- `const_copy_to(name, host, size)` — 将主机数据拷贝到命名的常量内存缓冲区，更新 KernelGlobals 中对应的指针，并同步到设备端

### 纹理管理

#### `tex_alloc(mem)`
- 2D 纹理：使用 SYCL 绑定式图像 API (`alloc_image_mem` + `create_image`) 创建 tile-optimized 存储
- 1D 纹理：使用线性设备内存 + 绑定式纹理句柄
- 支持多种通道格式：`unorm_int8`、`unorm_int16`、`fp32`、`fp16`
- 支持多种寻址模式：repeat、clamp_to_edge、clamp、mirrored_repeat
- NanoVDB 纹理跳过采样图像句柄创建，直接使用设备指针

#### `tex_copy_to(mem)`
- 2D 纹理使用 `ext_oneapi_copy`，1D 纹理使用 `generic_copy_to`

#### `tex_free(mem)`
- 释放绑定式纹理句柄 (`destroy_image_handle`) 和图像内存 (`free_image_mem`)

### BVH 构建

#### `build_bvh(bvh, progress, refit)` (条件编译 `WITH_EMBREE_GPU`)
- 当使用 `BVH_LAYOUT_EMBREEGPU` 时委托给 `BVHEmbree`
- 顶层 BVH 完成后获取遍历句柄并通过 `offload_scenes_to_gpu` 迁移到 GPU
- 其他情况回退到基类 `Device::build_bvh()`

#### `get_bvh_layout_mask(requested_features)`
- 硬件光线追踪可用时返回 `BVH_LAYOUT_EMBREEGPU`，否则返回 `BVH_LAYOUT_BVH2`

### 图形互操作

#### `should_use_graphics_interop(interop_device, log)`
- 仅支持 Vulkan 互操作（条件编译 `SYCL_LINEAR_MEMORY_INTEROP_AVAILABLE`）
- 通过 UUID 匹配验证 oneAPI 设备与 Vulkan 设备是否为同一物理设备

### 静态方法

#### `iterate_devices(cb, user_ptr)`
- 枚举所有可用 SYCL 设备，为每个设备调用回调函数，传递设备 ID、名称、硬件光线追踪支持、OIDN 支持和架构优化标志

#### `device_capabilities()`
- 返回所有设备的详细能力字符串，包含平台名称、架构、EU 数量、SIMD 宽度、全局内存大小、工作组大小、子组大小、USM 能力等

#### `architecture_information(device, name, is_optimized)`
- 查询设备架构枚举值，返回架构代号和是否经过 Intel/Blender 开发者优化
- 经过优化的架构包括：DG2（Arc Alchemist）、MTL、BMG、LNL、PTL 系列

### 辅助函数

#### `available_sycl_devices(multiple_dgpus_detected)` (文件作用域静态)
- 枚举所有 SYCL 平台和设备，过滤 OpenCL 后端
- 过滤条件：Intel Arc 及更新架构（EU > 96 或线程/EU != 7）、驱动版本满足最低要求
- 去重：通过 UUID 比较排除重复设备（解决 DPC++ 编译器在某些平台上重复报告设备的问题）
- 统计 Level-Zero dGPU 数量以检测多 dGPU 配置

#### `parse_driver_build_version(device)`
- 解析驱动版本字符串，支持 Linux（`xx.xx.xxxxx`）、Level-Zero（`x.x.xxxx`）和 Windows（`xx.xx.xxx.xxxx`）三种格式

#### `create_queue(external_queue, device_index, embree_device, multi_dgpu_flag)`
- 创建 SYCL 有序队列；多 dGPU 配置下为每个设备创建独立的 SYCL context 以避免兼容性问题
- 可选创建 Embree SYCL 设备

## 依赖关系

### 内部头文件
- `device/device.h` — 通用设备基类
- `device/oneapi/device.h` — 对外接口声明
- `device/oneapi/queue.h` — 设备队列类
- `kernel/device/oneapi/kernel.h` — oneAPI 内核接口
- `kernel/device/oneapi/globals.h` — `KernelGlobalsGPU` 定义
- `kernel/data_arrays.h` — 内核数据数组宏定义（通过 `#include` 展开）
- `session/display_driver.h` — 显示驱动接口（`GraphicsInteropDevice`）
- `bvh/embree.h` — Embree BVH 构建（条件编译）
- `util/log.h`, `util/map.h`, `util/unique_ptr.h`

### 被引用
- `src/device/oneapi/device.cpp` — 通过此头文件创建 `OneapiDevice` 实例
- `src/device/oneapi/queue.cpp` — 队列实现中访问 `OneapiDevice` 方法
- `src/device/oneapi/graphics_interop.cpp` — 图形互操作实现中访问设备方法
- `src/integrator/denoiser_oidn_gpu.cpp` — OIDN GPU 降噪器中访问设备信息

## 实现细节 / 关键算法

1. **USM 内存策略**: 选择 USM 设备内存（`sycl::malloc_device`）而非共享内存（`sycl::malloc_shared`），原因是 Cycles 架构已区分主机/设备指针并显式管理传输，共享内存的自动迁移机制反而会引入并发限制。调试模式（`WITH_ONEAPI_SYCL_HOST_TASK`）下切换为 `sycl::malloc_host`。

2. **惰性内存初始化对策**: `alloc_device()` 在分配后立即调用 `oneapi_zero_memory_on_device()` 进行零初始化，强制 GPU 运行时将分配落实到设备物理内存，避免惰性分配导致后续使用时的意外行为。

3. **多 dGPU 兼容性**: 检测到多块 Intel dGPU 时：(a) 为每个设备创建独立的 SYCL context；(b) 禁用 `prepare_for_device_copy` 优化。这些变通方案解决了 DPC++/Level-Zero 软件栈在多 dGPU 配置下的功能缺陷（参见 Blender issue #138384）。

4. **内存拷贝优化**: `usm_memcpy()` 区分四种情况：纯主机拷贝直接使用 `memcpy` 避免不必要的 SYCL 开销；设备到主机传输强制阻塞等待；其他传输异步执行。调试模式下所有操作均同步等待以获得更精确的错误信息。

5. **内核工作组大小调优**: 根据内核类型选择不同的工作组大小是 Intel GPU 性能优化的关键。SIMD8 设备（如某些 Arc 型号）的着色内核使用较小的 64 工作项以避免寄存器溢出。

6. **KernelGlobals 同步机制**: `const_copy_to()` 和 `global_alloc()` 在更新主机端 KernelGlobals 指针后，通过 `usm_memcpy` 整体同步到设备端副本 `kg_memory_device_`，确保设备内核访问到最新的数据指针。

7. **设备过滤与驱动版本检查**: `available_sycl_devices()` 通过 EU 数量/线程数筛选出 Arc 及更新架构，并验证驱动版本不低于最低支持版本（Windows: 101.8132 / Linux: NEO 34666），防止用户在不兼容的驱动上运行导致崩溃。

## 关联文件
- `src/device/oneapi/device.h` / `device.cpp` — 对外接口入口层
- `src/device/oneapi/queue.h` / `queue.cpp` — 设备队列实现
- `src/device/oneapi/graphics_interop.h` / `graphics_interop.cpp` — Vulkan 图形互操作
- `src/kernel/device/oneapi/kernel.h` — oneAPI 内核 DLL 接口
- `src/kernel/device/oneapi/globals.h` — `KernelGlobalsGPU` 结构体定义
- `src/bvh/embree.h` — Embree BVH 构建接口
