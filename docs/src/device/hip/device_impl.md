# device_impl.h / device_impl.cpp - HIP 设备核心实现

## 概述

本文件是 Cycles HIP 后端的核心实现，定义了 `HIPDevice` 类。该类继承自 `GPUDevice`，封装了 AMD GPU 上的完整设备生命周期管理，包括 HIP 上下文创建与销毁、内核编译与加载、各类内存（通用、全局、纹理、共享）的分配/拷贝/释放操作，以及图形互操作判断和设备队列创建。是整个 HIP 后端中最大、最核心的文件。

## 类与结构体

### `HIPDevice`
- **继承**: `GPUDevice`（`device/device.h` 中定义的 GPU 设备基类）
- **功能**: AMD HIP GPU 设备的完整抽象，管理单个 GPU 的上下文、模块、内核和内存操作
- **关键成员**:
  - `hipDevice_t hipDevice` — HIP 设备句柄
  - `hipCtx_t hipContext` — HIP 上下文句柄
  - `hipModule_t hipModule` — 已加载的 HIP 内核模块
  - `int pitch_alignment` — 纹理行对齐要求（字节数）
  - `int hipDevId` — HIP 设备序号
  - `int hipDevArchitecture` — GPU 架构版本号（major*100 + minor*10 格式）
  - `int hipRuntimeVersion` — HIP 运行时版本号
  - `bool first_error` — 首次错误标志，用于仅输出一次帮助文档链接
  - `HIPDeviceKernels kernels` — 内核函数缓存对象
- **关键方法**:
  - `HIPDevice()` — 构造函数，初始化 HIP 运行时、创建设备上下文、检测主机内存映射能力
  - `~HIPDevice()` — 析构函数，卸载模块并销毁上下文
  - `compile_kernel()` — 内核编译流程（优先预编译 fatbin，其次本地缓存，最后调用 HIPCC 编译）
  - `load_kernels()` — 加载内核模块到 GPU，初始化所有内核函数
  - `mem_alloc()` / `mem_copy_to()` / `mem_free()` 等 — 内存管理虚函数实现
  - `tex_alloc()` / `tex_free()` — 纹理对象创建与销毁
  - `gpu_queue_create()` — 创建 `HIPDeviceQueue` 实例

## 核心函数

### 构造与初始化

#### `HIPDevice::HIPDevice()`
- 调用 `hipInit(0)` 初始化 HIP 运行时
- 通过 `hipDeviceGet()` 获取设备句柄
- 检测 `hipDeviceAttributeCanMapHostMemory` 判断是否支持主机内存映射
- 读取 `hipDeviceAttributeTexturePitchAlignment` 获取纹理行对齐
- 使用 `hipCtxCreate()` 创建设备上下文，设置 `hipDeviceLmemResizeToMax`（预分配本地内存）和 `hipDeviceMapHost`（允许映射主机内存）标志
- 计算 GPU 架构版本号并获取 HIP 运行时版本
- 最后弹出上下文（`hipCtxPopCurrent`），由后续操作通过 `HIPContextScope` 按需推入

### 内核编译

#### `compile_kernel()`
- **编译优先级**: 预编译 fatbin(.fatbin.zst) > 本地缓存 > 调用 HIPCC 实时编译
- 使用源码 MD5 和编译标志的组合哈希作为缓存键
- VEGA (GCN 9.0) 架构降低优化级别为 `-O1` 以避免渲染瑕疵
- 编译选项包含 `-ffast-math -std=c++17`
- Windows 上若无预编译内核且不支持自适应编译则报错退出

#### `compile_kernel_get_common_cflags()`
- 生成通用编译标志，包括平台位数、include 路径
- 支持自适应编译时传入 `__KERNEL_FEATURES__` 宏
- 支持 `CYCLES_HIP_EXTRA_CFLAGS` 环境变量注入额外编译参数
- 条件性添加 `WITH_NANOVDB` 和 `WITH_CYCLES_DEBUG` 定义

#### `load_kernels()`
- 检查上下文有效性和设备支持
- 调用 `compile_kernel()` 获取 fatbin 路径
- 使用 `path_read_compressed_text()` 读取压缩格式的 fatbin 数据
- 通过 `hipModuleLoadData()` 加载到 GPU
- 成功后初始化内核缓存并预留本地内存

### 内存管理

#### `mem_alloc()` / `mem_copy_to()` / `mem_free()`
- 根据内存类型 (`MEM_TEXTURE`, `MEM_GLOBAL`, 通用) 分派到对应的具体实现
- 通用内存使用基类 `generic_alloc()` / `generic_copy_to()` / `generic_free()`
- 全局内存额外调用 `const_copy_to()` 更新内核参数中的指针

#### `tex_alloc()`
- 支持 1D（线性内存）和 2D（行对齐线性内存）两种纹理布局
- 为非 NanoVDB 纹理创建 `hipTextureObject_t` 无绑定纹理对象
- 配置寻址模式（Wrap/Clamp/Border/Mirror）和过滤模式（Point/Linear）
- 支持多种数据格式：UCHAR, UINT16, UINT, INT, FLOAT, HALF
- 通过 `texture_info` 数组管理纹理元数据槽位

#### `const_copy_to()`
- 通过 `hipModuleGetGlobal()` 获取内核参数全局变量 `kernel_params` 地址
- 使用 X-Macro 模式（`kernel/data_arrays.h`）匹配名称，更新对应数据指针

#### `reserve_local_memory()`
- 通过启动一个最小工作量的内核来触发 HIP 运行时预分配本地内存
- 选择最大的内核进行测试（根据 `KERNEL_FEATURE_NODE_RAYTRACE` 或 `KERNEL_FEATURE_MNEE` 选择）
- 记录预分配前后的可用内存差值

### 共享内存与设备内存

#### `shared_alloc()` / `shared_free()` / `shared_to_device_pointer()`
- 使用 `hipHostMalloc()` 分配映射的主机内存（`hipHostMallocMapped | hipHostMallocWriteCombined`）
- 通过 `hipHostGetDevicePointer()` 获取设备端指针

#### `alloc_device()` / `free_device()`
- 直接封装 `hipMalloc()` / `hipFree()`

### 其他功能

#### `should_use_graphics_interop()`
- 当前因 AMD 驱动 21.40 的 bug 以及缺少 Vulkan 支持，OpenGL 图形互操作始终返回 `false`
- headless 模式下直接返回 `false` 以避免驱动崩溃

#### `check_peer_access()`
- 检测两个 HIP 设备之间的 P2P 访问能力
- 同时检查数组访问支持（用于 3D 纹理）
- 双向启用对等访问

#### `get_bvh_layout_mask()`
- 返回 `BVH_LAYOUT_BVH2`，表示 HIP 设备使用 BVH2 布局（非硬件光追模式）

## 依赖关系
- **内部头文件**:
  - `device/device.h` — `GPUDevice` 基类
  - `device/hip/kernel.h` — `HIPDeviceKernels` 内核缓存
  - `device/hip/queue.h` — `HIPDeviceQueue` 设备队列
  - `device/hip/util.h` — `HIPContextScope` 上下文管理、错误检查宏
  - `kernel/device/hip/globals.h` — `KernelParamsHIP` 内核参数结构
  - `kernel/data_arrays.h` — 内核数据数组 X-Macro 定义
  - `session/display_driver.h` — 显示驱动接口
  - `util/debug.h`, `util/log.h`, `util/md5.h`, `util/path.h`, `util/string.h`, `util/system.h`, `util/time.h`, `util/types.h`
- **被引用**: `device/hip/device.cpp`、`device/hip/kernel.cpp`、`device/hip/queue.cpp`、`device/hip/graphics_interop.cpp`、`device/hip/util.cpp`、`device/hiprt/device_impl.cpp`、`device/hiprt/queue.cpp`

## 实现细节 / 关键算法

- **上下文管理模式**: HIP 上下文在构造后立即弹出，所有后续操作通过 `HIPContextScope` RAII 对象临时推入/弹出上下文，实现线程安全的上下文切换。
- **内核编译缓存策略**: 使用源码文件 MD5 + 编译标志的组合哈希作为缓存键，存储在 `path_cache_get("kernels/...")` 目录下。fatbin 文件使用 zstd 压缩格式（`.fatbin.zst`）。
- **纹理内存布局**: 2D 纹理使用行对齐 (`pitch_alignment`) 的线性内存而非 HIP 数组，以简化内存管理。对齐后的 pitch 通过 `align_up(src_pitch, pitch_alignment)` 计算。
- **内存回退机制**: 通过 `can_map_host` 标志和 `hipDeviceMapHost` 上下文标志，支持在 GPU 显存不足时将数据映射到主机内存。
- **自适应编译**: 通过 `DebugFlags().hip.adaptive_compile` 启用，编译时传入 `__KERNEL_FEATURES__` 宏，仅编译所需的着色特性，减少内核体积。

## 关联文件
- `src/device/hip/device.h` / `device.cpp` — 工厂入口
- `src/device/hip/kernel.h` / `kernel.cpp` — 内核函数加载
- `src/device/hip/queue.h` / `queue.cpp` — 命令队列
- `src/device/hip/graphics_interop.h` / `graphics_interop.cpp` — 图形互操作
- `src/device/hip/util.h` / `util.cpp` — 工具函数
- `src/device/hiprt/device_impl.h` — HIPRT 硬件光追设备（继承 `HIPDevice`）
- `src/kernel/device/hip/globals.h` — HIP 内核全局参数定义
