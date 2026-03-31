# device/hip - AMD HIP 设备实现

## 概述

`hip/` 子目录实现了 Cycles 的 AMD GPU 渲染后端。`HIPDevice` 继承自 `GPUDevice`，通过 HIP（Heterogeneous-computing Interface for Portability）API 管理 AMD GPU 的上下文、内存和内核执行。HIP API 在设计上与 CUDA Driver API 高度相似，因此 `HIPDevice` 的实现结构与 `CUDADevice` 基本一致。

该后端是 HIPRT 硬件光线追踪后端的基础——`HIPRTDevice` 进一步继承 `HIPDevice`。

核心特点：
- **波前路径追踪**：与 CUDA 后端类似，使用波前架构调度多个小内核。
- **RDNA 架构优化**：针对 AMD RDNA 系列架构进行优化。
- **动态编译**：支持运行时编译 HIP 内核。
- **P2P 显存访问**：支持多 AMD GPU 间直接内存访问。
- **图形互操作**：支持与 OpenGL/Vulkan 的图形资源互操作。

## 目录结构

| 文件名 | 类型 | 说明 |
|--------|------|------|
| `device.h` | 头文件 | `device_hip_init()`、`device_hip_create()`、`device_hip_info()`、`device_hip_capabilities()` 工厂函数 |
| `device.cpp` | 源文件 | HIP 设备枚举、初始化和能力查询 |
| `device_impl.h` | 头文件 | `HIPDevice` 类定义 |
| `device_impl.cpp` | 源文件 | `HIPDevice` 实现：上下文管理、内核编译/加载、内存与纹理管理 |
| `kernel.h` | 头文件 | `HIPDeviceKernel`、`HIPDeviceKernels` 内核缓存类 |
| `kernel.cpp` | 源文件 | HIP 内核函数查找与缓存 |
| `queue.h` | 头文件 | `HIPDeviceQueue` 命令队列类 |
| `queue.cpp` | 源文件 | HIP 流上的内核入队、同步和内存操作 |
| `util.h` | 头文件 | `HIPContextScope` 上下文管理、`hip_assert` 错误检查宏、架构检测辅助函数 |
| `util.cpp` | 源文件 | HIP 上下文管理实现 |
| `graphics_interop.h` | 头文件 | `HIPDeviceGraphicsInterop` 图形互操作类 |
| `graphics_interop.cpp` | 源文件 | OpenGL/Vulkan 资源的 HIP 映射实现 |

## 核心类与数据结构

### HIPDevice

定义位置：`device_impl.h`

继承自 `GPUDevice`。关键成员：

| 成员 | 说明 |
|------|------|
| `hipDevice` | HIP 设备句柄 (`hipDevice_t`) |
| `hipContext` | HIP 上下文 (`hipCtx_t`) |
| `hipModule` | 已加载的 HIP 模块 |
| `hipDevArchitecture` | GPU GCN/RDNA 架构版本 |
| `hipRuntimeVersion` | HIP 运行时版本 |
| `kernels` | `HIPDeviceKernels` 内核缓存 |

关键方法与 `CUDADevice` 对应方法功能一致：
- `load_kernels()` / `compile_kernel()` — 内核编译与加载
- `mem_alloc()` / `mem_free()` — 设备显存管理
- `tex_alloc()` / `tex_free()` — 纹理对象管理
- `check_peer_access()` — 多 GPU P2P 访问检查
- `gpu_queue_create()` — 创建 `HIPDeviceQueue`

### HIPContextScope

定义位置：`util.h`

RAII 风格的 HIP 上下文管理器。

### 辅助函数（util.h）

| 函数 | 说明 |
|------|------|
| `hipDeviceArch()` | 获取 GPU GCN 架构名称（如 `gfx1100`） |
| `hipSupportsDevice()` | 检查设备计算能力是否 >= 10（gfx10xx/RDNA） |
| `hipIsRDNA2OrNewer()` | 检查是否为 RDNA 2 或更新架构 |
| `hipSupportsDeviceOIDN()` | 检查设备是否支持 OpenImageDenoise（支持的架构：gfx1030、gfx1100-1102、gfx1200-1201） |
| `hipSupportsDriver()` | 检查 HIP 驱动兼容性 |

## 硬件要求

- **GPU**：AMD Radeon GPU，GCN 架构 >= gfx10xx（RDNA 及更新）
- **支持的架构**：
  - RDNA 1 (gfx1010, gfx1011, gfx1012)
  - RDNA 2 (gfx1030, gfx1031 等)
  - RDNA 3 (gfx1100, gfx1101, gfx1102)
  - RDNA 4 (gfx1200, gfx1201)
- **驱动**：AMD ROCm 兼容驱动（Linux）或 AMD Adrenalin 驱动（Windows）
- **HIP 运行时**：编译时需要 HIP SDK；运行时支持动态加载（`WITH_HIP_DYNLOAD` 使用 HIPEW）
- **降噪器**：OIDN 仅支持部分架构（gfx1030、gfx1100-1102、gfx1200-1201）

## API 封装

- **HIP API**：通过 `hipew.h`（HIP Extension Wrangler）动态加载或直接链接
- 核心 API 使用：`hipCtxCreate`、`hipModuleLoad`、`hipModuleLaunchKernel`、`hipMalloc`、`hipFree`、`hipStreamCreate` 等
- 错误处理：`hip_assert` / `hip_device_assert` 宏

## 依赖关系

### 上游依赖（本模块依赖）

| 模块 | 用途 |
|------|------|
| `device/` | `GPUDevice` 基类、`DeviceQueue` 基类、内存体系 |
| `kernel/` | 内核类型枚举 |
| `util/` | 字符串处理、日志、线程 |
| HIPEW / HIP SDK | HIP 运行时 API |

### 下游依赖（依赖本模块）

| 模块 | 用途 |
|------|------|
| `device/hiprt/` | `HIPRTDevice` 继承 `HIPDevice` |
| `device/device.cpp` | 工厂方法调用 `device_hip_create()` |

## 参见

- `src/device/hiprt/` — 基于 HIP 的 HIPRT 硬件光线追踪后端
- `src/device/cuda/` — 结构类似的 CUDA 后端
- `src/device/device.h` — `GPUDevice` 基类
- `src/kernel/device/hip/` — HIP 内核实现代码
