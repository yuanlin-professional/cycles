# device/cuda - NVIDIA CUDA 设备实现

## 概述

`cuda/` 子目录实现了 Cycles 的 NVIDIA CUDA GPU 渲染后端。`CUDADevice` 继承自 `GPUDevice`，通过 CUDA Driver API 管理 GPU 上下文、内存和内核执行。该后端是 OptiX 后端的基础——`OptiXDevice` 进一步继承 `CUDADevice`。

CUDA 后端的核心特点：
- **波前路径追踪**：使用波前（wavefront）架构将路径追踪拆分为多个小内核，通过 `CUDADeviceQueue` 按序调度以提高 GPU 利用率。
- **动态编译**：支持运行时编译 CUDA 内核（通过 NVRTC 或 nvcc），也支持使用预编译 PTX/cubin。
- **纹理对象**：使用 CUDA 纹理对象实现高效的纹理采样。
- **对等内存访问**：支持多 GPU 间 P2P 显存直接访问。
- **图形互操作**：支持与 OpenGL/Vulkan 的图形资源互操作，实现视口零拷贝渲染。

## 目录结构

| 文件名 | 类型 | 说明 |
|--------|------|------|
| `device.h` | 头文件 | `device_cuda_init()`、`device_cuda_create()`、`device_cuda_info()`、`device_cuda_capabilities()` 工厂函数 |
| `device.cpp` | 源文件 | CUDA 设备枚举、初始化和能力查询 |
| `device_impl.h` | 头文件 | `CUDADevice` 类定义 |
| `device_impl.cpp` | 源文件 | `CUDADevice` 实现：上下文创建、内核编译/加载、内存管理、纹理管理 |
| `kernel.h` | 头文件 | `CUDADeviceKernel`（单个内核信息）和 `CUDADeviceKernels`（内核缓存）类 |
| `kernel.cpp` | 源文件 | 从已加载 `CUmodule` 中查找并缓存所有内核函数 |
| `queue.h` | 头文件 | `CUDADeviceQueue` 命令队列类 |
| `queue.cpp` | 源文件 | CUDA 流上的内核入队、同步、内存操作 |
| `util.h` | 头文件 | `CUDAContextScope` 上下文管理、`cuda_assert` 错误检查宏、设备能力查询 |
| `util.cpp` | 源文件 | CUDA 上下文压栈/弹出实现 |
| `graphics_interop.h` | 头文件 | `CUDADeviceGraphicsInterop` 图形互操作类 |
| `graphics_interop.cpp` | 源文件 | OpenGL/Vulkan 资源的 CUDA 映射实现 |

## 核心类与数据结构

### CUDADevice

定义位置：`device_impl.h`

继承自 `GPUDevice`。关键成员：

| 成员 | 说明 |
|------|------|
| `cuDevice` | CUDA 设备句柄 (`CUdevice`) |
| `cuContext` | CUDA 上下文 (`CUcontext`) |
| `cuModule` | 已加载的 CUDA 模块（包含所有内核） |
| `cuDevArchitecture` | GPU 计算能力（如 86 表示 SM 8.6） |
| `kernels` | `CUDADeviceKernels` 内核缓存 |
| `pitch_alignment` | 纹理内存对齐要求 |

关键方法：

| 方法 | 说明 |
|------|------|
| `load_kernels()` | 编译或加载预编译内核到 `cuModule` |
| `compile_kernel()` | 调用 nvcc/NVRTC 编译 CUDA 源码为 PTX/cubin |
| `mem_alloc()` / `mem_free()` | 通过 `cuMemAlloc` / `cuMemFree` 管理设备显存 |
| `tex_alloc()` / `tex_free()` | CUDA 纹理对象创建与销毁 |
| `check_peer_access()` | 检查并启用多 GPU P2P 访问 |
| `gpu_queue_create()` | 创建 `CUDADeviceQueue` 实例 |
| `should_use_graphics_interop()` | 判断是否可使用图形互操作 |
| `reserve_local_memory()` | 为内核预留本地（共享）内存 |

### CUDADeviceKernel / CUDADeviceKernels

定义位置：`kernel.h`

- `CUDADeviceKernel`：单个 CUDA 内核的元信息，包含 `CUfunction` 函数指针、`num_threads_per_block`（占用率最优线程数）、`min_blocks`（最小块数）。
- `CUDADeviceKernels`：内核缓存，从已加载的 `CUmodule` 中通过名称查找所有 `DEVICE_KERNEL_NUM` 个内核。

### CUDADeviceQueue

定义位置：`queue.h`

继承自 `DeviceQueue`。基于 CUDA 流（`CUstream`）实现：
- `enqueue()` — 通过 `cuLaunchKernel` 在流上启动内核
- `synchronize()` — `cuStreamSynchronize` 等待完成
- `graphics_interop_create()` — 创建 CUDA 图形互操作对象

### CUDAContextScope

定义位置：`util.h`

RAII 风格的 CUDA 上下文管理器，构造时 `cuCtxPushCurrent`，析构时 `cuCtxPopCurrent`，确保多线程安全。

### CUDADeviceGraphicsInterop

定义位置：`graphics_interop.h`

实现 GPU 渲染结果直接写入 OpenGL/Vulkan 缓冲区。管理 `CUgraphicsResource`（OpenGL 互操作）和 `CUexternalMemory`（Vulkan 互操作）。

## 硬件要求

- **GPU**：NVIDIA GPU，Compute Capability >= 5.0（Maxwell 及更新架构）
- **驱动**：需要兼容的 NVIDIA 驱动，支持 CUDA Driver API
- **CUDA 工具包**：编译时需要 CUDA Toolkit；运行时支持动态加载（`WITH_CUDA_DYNLOAD` 使用 CUEW）
- **降噪器**：支持 OpenImageDenoise (OIDN)

## API 封装

- **CUDA Driver API**：通过 `cuew.h`（CUDA Extension Wrangler）动态加载或直接链接 `cuda.h`
- 核心 API 使用：`cuCtxCreate`、`cuModuleLoad`、`cuLaunchKernel`、`cuMemAlloc`、`cuMemFree`、`cuStreamCreate` 等
- 错误处理：`cuda_assert` / `cuda_device_assert` 宏

## 依赖关系

### 上游依赖（本模块依赖）

| 模块 | 用途 |
|------|------|
| `device/` | `GPUDevice` 基类、`DeviceQueue` 基类、内存体系 |
| `kernel/` | 内核类型枚举 |
| `util/` | 字符串处理、日志、线程 |
| CUEW / CUDA SDK | CUDA Driver API |

### 下游依赖（依赖本模块）

| 模块 | 用途 |
|------|------|
| `device/optix/` | `OptiXDevice` 继承 `CUDADevice` |
| `device/device.cpp` | 工厂方法调用 `device_cuda_create()` |

## 参见

- `src/device/optix/` — 基于 CUDA 的 OptiX 硬件光线追踪后端
- `src/device/device.h` — `GPUDevice` 基类
- `src/kernel/device/cuda/` — CUDA 内核实现代码
