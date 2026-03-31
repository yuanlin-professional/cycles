# compat.h - CUDA 设备兼容性层与类型抽象定义

## 概述

本文件为 Cycles 渲染器 CUDA 内核提供完整的兼容性抽象层。它定义了 GPU/CUDA 特性标识宏、设备函数限定符映射、线程/块/网格索引宏、空断言宏、GPU 纹理对象访问、快速数学函数替代，以及半精度浮点类型的内联汇编转换函数。该文件使 Cycles 内核代码能够以统一的 `ccl_*` 抽象接口编写，在 CUDA 平台上编译时映射为对应的 CUDA 原语。

## 核心函数/宏定义

### 平台标识宏
- **`__KERNEL_GPU__`**: 标记当前为 GPU 内核编译
- **`__KERNEL_CUDA__`**: 标记当前为 CUDA 平台
- **`CCL_NAMESPACE_BEGIN` / `CCL_NAMESPACE_END`**: 在 GPU 上定义为空（GPU 不使用命名空间）

### 设备函数限定符
| 抽象宏 | CUDA 映射 |
|--------|-----------|
| `ccl_device` | `__device__ __inline__` |
| `ccl_device_extern` | `extern "C" __device__` |
| `ccl_device_forceinline` | `__device__ __forceinline__` |
| `ccl_device_noinline` | `__device__ __noinline__` |
| `ccl_global` | (空) |
| `ccl_inline_constant` | `__constant__` |
| `ccl_device_constant` | `__constant__ __device__` |
| `ccl_gpu_shared` | `__shared__` |
| `ccl_restrict` | `__restrict__` |
| `ccl_align(n)` | `__align__(n)` |

### `kernel_assert(cond)`
定义为空操作，CUDA 不支持设备端断言。

### GPU 线程索引宏
- **`ccl_gpu_thread_idx_x`**: `threadIdx.x`
- **`ccl_gpu_block_dim_x`**: `blockDim.x`
- **`ccl_gpu_block_idx_x`**: `blockIdx.x`
- **`ccl_gpu_grid_dim_x`**: `gridDim.x`
- **`ccl_gpu_warp_size`**: `warpSize`
- **`ccl_gpu_global_id_x()`**: 计算全局线程 ID
- **`ccl_gpu_global_size_x()`**: 计算全局线程总数
- **`ccl_gpu_thread_mask(thread_warp)`**: 生成线程掩码（`0xFFFFFFFF >> (warpSize - thread_warp)`）

### GPU 同步
- **`ccl_gpu_syncthreads()`**: `__syncthreads()`
- **`ccl_gpu_ballot(predicate)`**: `__ballot_sync(0xFFFFFFFF, predicate)`

### 纹理对象
- **`ccl_gpu_tex_object_2D`**: 类型为 `CUtexObject`（`unsigned long long`）
- **`ccl_gpu_tex_object_read_2D<T>(texobj, x, y)`**: 模板函数，调用 `tex2D<T>` 进行硬件纹理采样

### 快速数学函数
将标准数学函数替换为 CUDA 内建快速版本（精度略低但性能更高）：
- `cosf` -> `__cosf`、`sinf` -> `__sinf`、`powf` -> `__powf`
- `tanf` -> `__tanf`、`logf` -> `__logf`、`expf` -> `__expf`

### 半精度浮点
- **`half`**: 定义为 `unsigned short`
- **`__float2half(f)`**: 使用 PTX 内联汇编 `cvt.rn.f16.f32` 转换
- **`__half2float(h)`**: 使用 PTX 内联汇编 `cvt.f32.f16` 转换

### RTC 编译兼容
- `__CUDACC_RTC__` 模式下手动定义 `uint32_t` 和 `uint64_t`
- `CYCLES_CUBIN_CC` 模式下定义 `FLT_MIN`、`FLT_MAX`、`FLT_EPSILON`

## 依赖关系

- **内部头文件**:
  - `util/half.h` (半精度浮点工具)
  - `util/types.h` (Cycles 类型系统)
- **被引用**: `src/kernel/device/cuda/kernel.cu`

## 实现细节 / 关键算法

### 统一抽象层设计
Cycles 内核代码使用 `ccl_*` 前缀的抽象宏编写，每个设备后端（CPU、CUDA、HIP、Metal、oneAPI）提供各自的 `compat.h` 映射。这使得内核主体代码可以完全设备无关，仅通过兼容层适配不同平台。

### PTX 内联汇编
半精度浮点转换使用 NVIDIA PTX 指令集的内联汇编，直接映射到 GPU 硬件的类型转换指令，比软件实现更高效。

## 关联文件

- `src/kernel/device/cpu/compat.h` — CPU 兼容层（`kernel_assert` 为真实断言）
- `src/kernel/device/hip/compat.h` — HIP 兼容层（结构几乎相同，差异在投票和纹理 API）
- `src/kernel/device/cuda/config.h` — CUDA 内核启动配置
- `src/kernel/device/cuda/globals.h` — CUDA 全局数据
