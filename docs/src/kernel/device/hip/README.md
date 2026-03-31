# kernel/device/hip - HIP 内核编译入口

## 概述

`kernel/device/hip/` 是 Cycles 路径追踪引擎针对 AMD GPU（通过 HIP/ROCm 平台）的内核编译入口。其结构与 CUDA 后端高度对称：`kernel.cpp` 在 `__HIP_DEVICE_COMPILE__` 宏保护下引入兼容性定义和全局变量，然后包含 GPU 通用内核代码。

HIP（Heterogeneous-computing Interface for Portability）是 AMD 的 GPU 编程接口，在 API 设计上与 CUDA 极为相似。Cycles 的 HIP 后端利用这种相似性，通过几乎相同的宏映射来复用 GPU 通用内核代码。HIP 编译器（hipcc）可将同一份源码编译为在 AMD GPU 上运行的二进制代码。

与 CUDA 不同，HIP 后端默认使用更大的线程 block 大小（1024），以充分利用 AMD GPU 架构中更多的 compute unit 和更大的 wavefront。

## 目录结构

| 文件名 | 类型 | 说明 |
|--------|------|------|
| `kernel.cpp` | 源文件 | HIP 内核编译入口。在 `__HIP_DEVICE_COMPILE__` 宏保护下，引入 `compat.h`、`config.h`、`globals.h`，以及 GPU 通用的 `image.h` 和 `kernel.h`。文件扩展名为 `.cpp`（而非 `.cu`），因为 HIP 使用 C++ 编译器前端 |
| `compat.h` | 头文件 | HIP 兼容性定义。定义 `__KERNEL_GPU__` 和 `__KERNEL_HIP__` 宏；将 Cycles 抽象限定符映射为 HIP 原语；包含 `hip/hip_fp16.h` 和 `hip/hip_runtime.h`；使用 HIP 快速数学函数；定义 64 位线程掩码（与 CUDA 的 32 位不同，AMD wavefront 为 64 线程） |
| `config.h` | 头文件 | HIP 设备配置和启动边界。定义统一的线程 block 大小 1024、最大寄存器 64；为 HIPRT 内核额外定义 `GPU_HIPRT_KERNEL_BLOCK_NUM_THREADS=1024`；每多处理器最大 block 数为 64（CUDA 为 32） |
| `globals.h` | 头文件 | HIP 全局数据结构。定义 `KernelParamsHIP` 结构体（与 CUDA 版本结构相同），存储于 `__constant__` 内存中。`KernelGlobals` 为空结构体指针 |

## 内核函数入口

所有内核函数在 `kernel/device/gpu/kernel.h` 中定义，通过 `ccl_gpu_kernel_signature` 宏展开为：

```c
extern "C" __global__ void __launch_bounds__(block_num_threads, min_blocks)
    kernel_gpu_<name>(...)
```

HIP 后端直接复用全部 GPU 通用内核，无平台特有内核。主要内核函数参见 `kernel/device/gpu/` 文档。

## GPU 兼容性

**支持的 AMD GPU 架构：**

HIP 后端不像 CUDA 那样按计算能力分别配置，而是使用统一的启动边界参数：

| 参数 | 值 | 说明 |
|------|-----|------|
| 每 block 线程数 | 1024 | 适配 AMD 大 wavefront |
| 每线程最大寄存器 | 64 | |
| 每多处理器最大寄存器 | 65536 | |
| 每多处理器最大 block | 64 | 高于 CUDA 的 32 |
| 每 block 最大线程 | 1024 | |
| Wavefront 大小 | 64 | AMD 默认（CUDA warp 为 32） |

**编译器要求：**
- hipcc（ROCm Toolkit）
- 支持 `__HIPCC_RTC__` 运行时编译
- 支持 `CYCLES_HIPBIN_CC` 自定义编译模式
- 在 Windows 上需要 MSVC `immintrin.h`

**线程掩码差异：**
HIP 使用 64 位线程掩码 `uint64_t((1ull << thread_warp) - 1)`，而 CUDA 使用 32 位 `uint(0xFFFFFFFF >> (warpSize - thread_warp))`。这是因为 AMD GPU 的 wavefront 默认为 64 线程。

## API 封装

底层使用 **HIP Runtime API**：

| Cycles 抽象 | HIP 实现 |
|-------------|----------|
| `ccl_device` | `__device__ __inline__` |
| `ccl_global` | （空） |
| `ccl_gpu_shared` | `__shared__` |
| `ccl_inline_constant` | `__constant__` |
| `ccl_gpu_syncthreads()` | `__syncthreads()` |
| `ccl_gpu_ballot(pred)` | `__ballot(pred)` |
| `ccl_gpu_thread_idx_x` | `threadIdx.x` |
| `ccl_gpu_block_dim_x` | `blockDim.x` |
| `ccl_gpu_tex_object_read_2D<T>` | `tex2D<T>(texobj, x, y)` |
| 快速数学 | `__cosf`, `__sinf`, `__powf`, `__logf`, `__expf`, `__tanf` |

注意 HIP 的 `__ballot()` 不需要 `sync` 参数（与 CUDA 的 `__ballot_sync` 不同）。

纹理使用 `hipTextureObject_t` 类型。

## 依赖关系

### 上游依赖（本模块依赖）
- `kernel/device/gpu/kernel.h` -- GPU 通用内核定义
- `kernel/device/gpu/image.h` -- GPU 纹理采样
- `kernel/types.h` -- 内核类型定义
- `kernel/integrator/state.h` -- 积分器状态
- `kernel/data_arrays.h` -- 数据数组定义
- `hip/hip_runtime.h`, `hip/hip_fp16.h` -- HIP 运行时和半精度支持
- `util/types.h`, `util/half.h`, `util/color.h`, `util/texture.h` -- 基础类型

### 下游依赖（依赖本模块）
- `device/hip/` -- HIP 设备主机端实现
- `kernel/device/hiprt/` -- HIPRT 后端复用 `compat.h` 和 `config.h`

## 参见

- `kernel/device/gpu/` -- GPU 通用内核基础设施
- `kernel/device/hiprt/` -- HIPRT 后端（使用 AMD 硬件光线追踪）
- `kernel/device/cuda/` -- CUDA 后端（结构对称）
- `device/hip/` -- HIP 设备主机端代码
