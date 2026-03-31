# device/metal - Apple Metal 设备实现

## 概述

`metal/` 子目录实现了 Cycles 的 Apple Metal GPU 渲染后端。`MetalDevice` 直接继承 `Device` 基类（而非 `GPUDevice`），拥有完全独立的内存管理和内核编译体系。该后端针对 Apple Silicon（M1/M2/M3 系列）和支持 Metal 的 AMD GPU 优化。

核心特点：
- **MetalRT 硬件光线追踪**：在 Apple Silicon 上可选使用 Metal Ray Tracing API 加速光线求交，构建 Metal 加速结构（BLAS/TLAS）。
- **三级管线特化**：定义了三种 Pipeline State Object (PSO) 类型，从快速通用到深度特化，平衡编译时间与运行时性能。
- **Metal Function Constants**：使用 Metal 函数常量替换 `KernelData` 变量，实现编译时常量折叠优化。
- **二进制归档缓存**：支持将编译后的管线状态序列化到磁盘，加速后续渲染启动。
- **原子排序**：支持本地原子排序内核以优化波前调度。
- **GPU Trace 捕获**：支持 Metal GPU 跟踪捕获用于性能分析。

## 目录结构

| 文件名 | 类型 | 说明 |
|--------|------|------|
| `device.h` | 头文件 | `device_metal_init()`、`device_metal_exit()`、`device_metal_create()`、`device_metal_info()`、`device_metal_capabilities()` |
| `device.mm` | 源文件 | Metal 设备枚举与初始化（Objective-C++） |
| `device_impl.h` | 头文件 | `MetalDevice` 类定义 |
| `device_impl.mm` | 源文件 | `MetalDevice` 实现：设备初始化、内存管理、内核加载、BVH 构建 |
| `kernel.h` | 头文件 | `MetalKernelPipeline`、`MetalDispatchPipeline`、`MetalDeviceKernels` 命名空间、PSO 类型枚举 |
| `kernel.mm` | 源文件 | Metal 计算管线编译、缓存管理、二进制归档 |
| `queue.h` | 头文件 | `MetalDeviceQueue` 命令队列类 |
| `queue.mm` | 源文件 | Metal 命令缓冲区与计算编码器管理、内核调度、性能分析 |
| `bvh.h` | 头文件 | `BVHMetal` 加速结构类 |
| `bvh.mm` | 源文件 | Metal 加速结构构建（BLAS/TLAS） |
| `util.h` | 头文件 | `MetalInfo` 静态辅助函数、`AppleGPUArchitecture` 枚举、GPU 地址辅助函数 |
| `util.mm` | 源文件 | Metal 设备信息查询、架构检测 |
| `graphics_interop.h` | 头文件 | `MetalDeviceGraphicsInterop` 图形互操作类 |
| `graphics_interop.mm` | 源文件 | Metal 与图形 API 互操作实现 |

## 核心类与数据结构

### MetalDevice

定义位置：`device_impl.h`

继承自 `Device`（不经过 `GPUDevice`）。关键成员：

| 成员 | 说明 |
|------|------|
| `mtlDevice` | Metal 设备对象 (`id<MTLDevice>`) |
| `mtlLibrary[PSO_NUM]` | 编译后的 Metal 库（每种 PSO 类型一个） |
| `mtlComputeCommandQueue` | 计算命令队列 |
| `mtlGeneralCommandQueue` | 通用命令队列 |
| `launch_params_buffer` | 内核启动参数缓冲区 |
| `launch_params` | 内核参数指针 (`KernelParamsMetal*`) |
| `use_metalrt` | 是否启用 MetalRT 硬件光线追踪 |
| `use_metalrt_extended_limits` | 是否使用扩展 MetalRT 限制 |
| `blas_buffer` / `blas_array` / `accel_struct` | MetalRT 加速结构 |
| `metal_mem_map` | Metal 内存分配映射表 |
| `texture_info` / `texture_bindings` / `texture_slot_map` | 绑定式纹理管理 |
| `kernel_specialization_level` | 当前使用的管线特化级别 |

### MetalPipelineType 枚举（PSO 类型）

定义位置：`kernel.h`

| 类型 | 说明 |
|------|------|
| `PSO_GENERIC` | 通用管线——支持所有功能，编译慢但只需一次，渲染可立即开始 |
| `PSO_SPECIALIZED_INTERSECT` | 特化求交管线——使用函数常量优化求交内核，编译较快 |
| `PSO_SPECIALIZED_SHADE` | 特化着色管线——使用函数常量优化着色内核并剔除未使用的 SVM 节点，编译慢但性能最佳 |

### MetalKernelPipeline

定义位置：`kernel.h`

单个 Metal 计算管线状态。可在多个 `MetalDeviceQueue` 实例间共享。关键字段：
- `pipeline` — `id<MTLComputePipelineState>`
- `device_kernel` — 对应的设备内核类型
- `pso_type` — PSO 类型
- `table_functions` — MetalRT 交叉函数表

### MetalDispatchPipeline

定义位置：`kernel.h`

管线实例（与特定 `MetalDeviceQueue` 绑定），管理交叉函数表实例。

### MetalDeviceQueue

定义位置：`queue.h`

继承自 `DeviceQueue`。基于 Metal 命令缓冲区和计算编码器实现：
- 使用 `id<MTLCommandBuffer>` 和 `id<MTLComputeCommandEncoder>` 调度内核
- 支持 `MTLSharedEvent` 实现 CPU-GPU 同步
- 内置 GPU 性能分析支持（`CYCLES_METAL_PROFILING`）
- 支持 `.gputrace` 捕获（`CYCLES_DEBUG_METAL_CAPTURE_*`）
- `supports_local_atomic_sort()` 返回 true（在支持的设备上）

### BVHMetal

定义位置：`bvh.h`

Metal 加速结构实现，继承自 `BVH`。支持：
- `build_BLAS_mesh()` — 三角形网格 BLAS
- `build_BLAS_hair()` — 毛发 BLAS
- `build_BLAS_pointcloud()` — 点云 BLAS
- `build_TLAS()` — 顶层加速结构
- Per-Component Motion Interpolation（macOS 15+，`use_pcmi`）
- 扩展限制（`extended_limits`）

### AppleGPUArchitecture 枚举

定义位置：`util.h`

```
APPLE_M1, APPLE_M2, APPLE_M2_BIG, APPLE_M3, APPLE_UNKNOWN
```

用于根据 Apple Silicon 代数选择最优配置参数。

## 硬件要求

- **平台**：macOS 11.0 (Big Sur) 及更新
- **GPU**：
  - Apple Silicon（M1、M2、M3 系列）——完整支持，包括 MetalRT
  - AMD GPU（macOS 上的独立显卡）——基本支持
- **MetalRT**：需要 Apple Silicon，macOS 11.0+
- **降噪器**：支持 OpenImageDenoise (OIDN)
- **条件编译**：`WITH_METAL`

## API 封装

- **Metal API**（Objective-C++）：
  - 设备管理：`MTLCreateSystemDefaultDevice`、`MTLCopyAllDevices`
  - 编译：`MTLLibrary`、`MTLComputePipelineState`、`MTLFunction`
  - 内存：`MTLBuffer`、`MTLTexture`
  - 命令：`MTLCommandQueue`、`MTLCommandBuffer`、`MTLComputeCommandEncoder`
  - 加速结构：`MTLAccelerationStructure`（MetalRT）
  - 同步：`MTLSharedEvent`

## 依赖关系

### 上游依赖（本模块依赖）

| 模块 | 用途 |
|------|------|
| `device/` | `Device` 基类、`DeviceQueue` 基类、内存体系 |
| `bvh/` | BVH 参数、BVH 基类 |
| `kernel/` | 内核类型枚举、Metal 内核参数 (`KernelParamsMetal`) |
| `util/` | 线程、日志 |
| Metal Framework | Apple Metal GPU API |

### 下游依赖（依赖本模块）

| 模块 | 用途 |
|------|------|
| `device/device.cpp` | 工厂方法调用 `device_metal_create()` |

## 参见

- `src/device/device.h` — `Device` 基类
- `src/device/optix/` — 类似的硬件光线追踪实现（NVIDIA）
- `src/device/hiprt/` — 类似的硬件光线追踪实现（AMD）
- `src/kernel/device/metal/` — Metal 内核实现
