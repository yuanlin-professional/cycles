# device_impl.h / device_impl.cpp - CPU 设备的完整实现

## 概述

本文件定义并实现了 `CPUDevice` 类，它是 Cycles 渲染引擎中 CPU 渲染设备的核心实现。`CPUDevice` 继承自 `Device` 基类，负责 CPU 端的内存管理、纹理分配、层次包围体(BVH)构建、路径引导设备管理以及内核线程全局状态的初始化。该类是 CPU 渲染管线中设备层的枢纽。

## 类与结构体

### CPUDevice

- **继承**: `Device`（公有继承）
- **功能**: 作为 CPU 渲染后端的设备抽象实现，管理 CPU 侧所有渲染资源的生命周期，包括内存、纹理、层次包围体(BVH)和内核全局状态。

#### 关键成员

| 成员 | 类型 | 说明 |
|------|------|------|
| `kernel_globals` | `KernelGlobalsCPU` | CPU 内核的全局数据，存储渲染所需的全部常量与数据指针 |
| `texture_info` | `device_vector<TextureInfo>` | 纹理信息向量，以槽位索引管理所有纹理的元数据 |
| `need_texture_info` | `bool` | 标记纹理信息是否需要刷新到设备 |
| `osl_globals` | `OSLGlobals` | Open Shading Language 全局数据（仅在 `WITH_OSL` 编译时存在） |
| `embree_traversable` | `RTCTraversable` / `RTCScene` | Embree 光线遍历句柄，指向顶层 BVH 场景（仅在 `WITH_EMBREE` 编译时存在） |
| `embree_device` | `RTCDevice` | Embree 设备实例（仅在 `WITH_EMBREE` 编译时存在） |
| `guiding_device` | `unique_ptr<openpgl::cpp::Device>` | OpenPGL 路径引导设备，惰性初始化（仅在 `WITH_PATH_GUIDING` 编译时存在） |

#### 关键方法

| 方法 | 说明 |
|------|------|
| `CPUDevice()` | 构造函数。初始化 `texture_info`，创建 Embree 设备，从 `TaskScheduler` 获取最大并发线程数 |
| `~CPUDevice()` | 析构函数。释放 Embree 设备，释放纹理信息 |
| `get_bvh_layout_mask()` | 返回支持的 BVH 布局掩码（`BVH_LAYOUT_BVH2`，可选 `BVH_LAYOUT_EMBREE`） |
| `load_texture_info()` | 若纹理信息已更新，将其拷贝到设备端。返回是否执行了拷贝 |
| `mem_alloc()` | 分配设备内存。对 `MEM_DEVICE_ONLY` 类型使用对齐内存分配，其他类型直接映射主机指针 |
| `mem_copy_to()` | 将内存拷贝到设备。对全局内存和纹理分别调用 `global_alloc` / `tex_alloc`，普通内存为空操作 |
| `mem_move_to_host()` | 空操作（CPU 设备内存即主机内存） |
| `mem_copy_from()` | 空操作（CPU 设备内存即主机内存） |
| `mem_zero()` | 将设备内存清零（`memset`） |
| `mem_free()` | 释放设备内存，根据类型分派到 `global_free`、`tex_free` 或直接释放 |
| `mem_alloc_sub_ptr()` | 返回设备内存中指定偏移量处的子指针 |
| `const_copy_to()` | 将常量数据拷贝到内核全局状态。若启用 Embree，会更新 `KernelData` 中的 BVH 遍历句柄 |
| `global_alloc()` | 分配全局内存并注册到 `kernel_globals` |
| `global_free()` | 释放全局内存 |
| `tex_alloc()` | 分配纹理内存，更新纹理信息槽位（按需扩展，步长 128） |
| `tex_free()` | 释放纹理内存并标记纹理信息需要刷新 |
| `build_bvh()` | 构建层次包围体(BVH)。优先使用 Embree 构建（支持 refit 和 build），否则回退到基类实现 |
| `get_guiding_device()` | 惰性创建并返回 OpenPGL 路径引导设备（根据 CPU 支持选择 4 或 8 宽度） |
| `get_cpu_kernel_thread_globals()` | 为每个 CPU 线程创建独立的 `ThreadKernelGlobalsCPU`，确保纹理信息已加载 |
| `get_cpu_osl_memory()` | 返回 OSL 全局数据指针（未启用 OSL 时返回 `nullptr`） |
| `load_kernels()` | 加载内核（CPU 内核始终可用，直接返回 `true`） |

## 核心函数

本文件的所有核心逻辑均封装在 `CPUDevice` 类的方法中（见上方表格）。

## 依赖关系

### 内部头文件
- `device/cpu/device_impl.h` — 本类头文件
- `device/cpu/kernel.h` — `CPUKernels` 类，用于获取内核函数
- `device/device.h` — `Device` 基类
- `device/memory.h` — `device_memory`、`device_texture` 等内存抽象
- `kernel/device/cpu/kernel.h` — CPU 端内核入口函数声明（`kernel_const_copy`、`kernel_global_memory_copy`）
- `kernel/globals.h` — `KernelGlobalsCPU`、`ThreadKernelGlobalsCPU` 定义
- `kernel/osl/globals.h` — `OSLGlobals` 定义
- `kernel/types.h` — `KernelData` 等内核数据类型
- `bvh/embree.h` — `BVHEmbree` 类（Embree BVH 封装）
- `session/buffers.h` — 渲染缓冲区
- `util/guiding.h` — 路径引导设备类型检测
- `util/log.h`、`util/progress.h`、`util/task.h`、`util/unique_ptr.h` — 工具类

### 被引用
- `src/device/cpu/device.cpp` — 工厂函数中 `#include "device/cpu/device_impl.h"` 以实例化 `CPUDevice`
- `src/scene/geometry_bvh.cpp` — 几何体 BVH 构建中引用 `CPUDevice`

## 实现细节 / 关键算法

### 内存管理策略

CPU 设备的内存管理本质上是对主机内存的直接映射：
- **普通内存**: `device_pointer` 直接指向 `host_pointer`，拷贝操作为空操作（no-op），因为 CPU 设备与主机共享同一地址空间。
- **仅设备内存** (`MEM_DEVICE_ONLY`): 使用 `util_aligned_malloc` 分配对齐内存（满足 `MIN_ALIGNMENT_DEVICE_MEMORY` 要求）。
- **全局内存**: 通过 `kernel_global_memory_copy` 注册到内核全局状态。
- **纹理内存**: 分配后更新 `texture_info` 槽位表，槽位表按需以 128 为步长扩展，减少频繁重分配。

### BVH 构建

`build_bvh()` 方法优先检查 Embree 布局（包括多设备混合布局如 `BVH_LAYOUT_MULTI_OPTIX_EMBREE` 等），使用 `BVHEmbree` 进行构建或 refit。若为顶层 BVH，会将 Embree 场景的遍历句柄存储到 `embree_traversable`，供后续光线追踪使用。Embree 4.4+ 版本使用 `RTCTraversable`，旧版本使用 `RTCScene`。

### 路径引导设备

`get_guiding_device()` 采用惰性初始化模式，仅在首次调用时创建 OpenPGL 设备。根据 `guiding_device_type()` 返回的 SIMD 宽度（4 或 8），选择对应的 PGL 设备类型（`PGL_DEVICE_TYPE_CPU_4` 或 `PGL_DEVICE_TYPE_CPU_8`）。

### 线程内核全局状态

`get_cpu_kernel_thread_globals()` 为每个 CPU 线程创建独立的 `ThreadKernelGlobalsCPU` 副本。在创建前调用 `load_texture_info()` 确保纹理数据已同步。每个线程的全局状态包含对 `kernel_globals`、OSL 全局数据和性能分析器的引用。

## 关联文件

- `src/device/cpu/device.h` / `device.cpp` — CPU 设备工厂入口，实例化本类
- `src/device/cpu/kernel.h` / `kernel.cpp` — CPU 内核函数集合
- `src/device/device.h` — `Device` 基类定义
- `src/kernel/device/cpu/kernel.h` — CPU 端底层内核函数
- `src/bvh/embree.h` — Embree 层次包围体(BVH)封装
- `src/integrator/path_trace_work_cpu.cpp` — CPU 路径追踪工作调度，使用本设备提供的内核和线程全局状态
