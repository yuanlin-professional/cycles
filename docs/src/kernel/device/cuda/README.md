# kernel/device/cuda - CUDA 内核编译入口

## 概述

`kernel/device/cuda/` 是 Cycles 路径追踪引擎针对 NVIDIA CUDA GPU 的内核编译入口。该目录结构极为精简：`kernel.cu` 仅配置 CUDA 特定的兼容性定义和全局变量布局，然后直接引入 `kernel/device/gpu/` 中的通用 GPU 内核代码。

CUDA 内核通过 NVCC 编译器编译为 PTX 或 cubin 文件，采用波前路径追踪架构。所有内核函数以 `extern "C" __global__` 形式导出，通过 `__launch_bounds__` 指令控制线程 block 大小和寄存器使用量，以优化占用率。场景数据通过 `__constant__` 内存空间中的 `KernelParamsCUDA` 结构体传递。

## 目录结构

| 文件名 | 类型 | 说明 |
|--------|------|------|
| `kernel.cu` | 源文件 | CUDA 内核编译入口。在 `__CUDA_ARCH__` 宏保护下，依次引入 `compat.h`、`config.h`、`globals.h`，以及 GPU 通用的 `image.h` 和 `kernel.h` |
| `compat.h` | 头文件 | CUDA 兼容性定义。定义 `__KERNEL_GPU__` 和 `__KERNEL_CUDA__` 宏；将 Cycles 抽象限定符（`ccl_device`、`ccl_global`、`ccl_gpu_shared` 等）映射为 CUDA 原语（`__device__`、`__shared__` 等）；定义线程索引宏（`threadIdx.x`、`blockDim.x` 等）；使用 CUDA 快速数学函数（`__cosf`、`__sinf` 等）；通过 PTX 内联汇编实现半精度浮点转换 |
| `config.h` | 头文件 | CUDA 设备配置和启动边界。根据计算能力（Compute Capability）定义线程 block 大小和最大寄存器数：CC 5.x-6.x 使用 256 线程/48-64 寄存器；CC 7.x-8.x-12.x 使用 384 线程/168 寄存器。定义 `ccl_gpu_kernel` 和 `ccl_gpu_kernel_signature` 宏 |
| `globals.h` | 头文件 | CUDA 全局数据结构。定义 `KernelParamsCUDA` 结构体（包含 `KernelData`、数据数组指针和 `IntegratorStateGPU`），存储于 `__constant__` 内存中。`KernelGlobals` 定义为空结构体指针（期望被编译器优化掉） |

## 内核函数入口

所有内核函数在 `kernel/device/gpu/kernel.h` 中定义，通过 `ccl_gpu_kernel_signature` 宏展开为：

```c
extern "C" __global__ void __launch_bounds__(block_num_threads, min_blocks)
    kernel_gpu_<name>(...)
```

主要内核函数参见 `kernel/device/gpu/` 文档。CUDA 后端直接复用全部 GPU 通用内核，无平台特有内核。

## GPU 兼容性

**支持的计算能力（Compute Capability）：**

| 计算能力 | 架构 | 线程/Block | 最大寄存器/线程 | 说明 |
|----------|------|-----------|----------------|------|
| 5.x (Maxwell) | SM50-SM52 | 256 | 48 | 最低支持版本 |
| 6.x (Pascal) | SM60-SM62 | 256 | 64 | CUDA 9.0+ 使用 64 寄存器 |
| 7.x (Volta/Turing) | SM70-SM75 | 384 | 168 | |
| 8.x (Ampere) | SM80-SM89 | 384 | 168 | |
| 12.x (Blackwell) | SM120 | 384 | 168 | |

**编译器要求：**
- NVCC（CUDA Toolkit）或 NVRTC（运行时编译）
- 支持 `__CUDACC_RTC__` 用于运行时编译（无需标准库头文件）
- 支持 `CYCLES_CUBIN_CC` 自定义编译模式

**硬件约束：**
- 每多处理器最大寄存器数: 65536
- 每多处理器最大 block 数: 32
- 每 block 最大线程数: 1024
- 每线程最大寄存器数: 255
- Warp 大小: 32（通过 `warpSize` 内置变量获取）

## API 封装

底层使用 **CUDA Runtime / Driver API**：

| Cycles 抽象 | CUDA 实现 |
|-------------|-----------|
| `ccl_device` | `__device__ __inline__` |
| `ccl_global` | （空，全局内存为默认） |
| `ccl_gpu_shared` | `__shared__` |
| `ccl_inline_constant` | `__constant__` |
| `ccl_gpu_syncthreads()` | `__syncthreads()` |
| `ccl_gpu_ballot(pred)` | `__ballot_sync(0xFFFFFFFF, pred)` |
| `ccl_gpu_thread_idx_x` | `threadIdx.x` |
| `ccl_gpu_block_dim_x` | `blockDim.x` |
| `ccl_gpu_tex_object_read_2D<T>` | `tex2D<T>(texobj, x, y)` |
| 快速数学 | `__cosf`, `__sinf`, `__powf`, `__logf`, `__expf`, `__tanf` |

纹理使用 `CUtexObject`（`unsigned long long`）类型的绑定纹理对象。

## 依赖关系

### 上游依赖（本模块依赖）
- `kernel/device/gpu/kernel.h` -- GPU 通用内核定义
- `kernel/device/gpu/image.h` -- GPU 纹理采样
- `kernel/types.h` -- 内核类型定义
- `kernel/integrator/state.h` -- 积分器状态
- `kernel/data_arrays.h` -- 数据数组定义
- `util/types.h`, `util/half.h`, `util/color.h`, `util/texture.h` -- 基础类型和工具

### 下游依赖（依赖本模块）
- `device/cuda/` -- CUDA 设备主机端实现，负责加载和调度 CUDA 内核

## 参见

- `kernel/device/gpu/` -- GPU 通用内核基础设施
- `kernel/device/optix/` -- OptiX 后端（同为 NVIDIA 平台，使用硬件光线追踪）
- `device/cuda/` -- CUDA 设备主机端代码
