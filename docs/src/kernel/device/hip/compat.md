# compat.h - HIP 设备兼容性层与类型抽象定义

## 概述

本文件为 Cycles 渲染器 HIP（AMD GPU）内核提供完整的兼容性抽象层。其结构和功能与 CUDA 版本的 `compat.h` 高度相似，定义了 GPU/HIP 特性标识宏、设备函数限定符映射、线程索引宏、空断言宏、纹理对象访问和快速数学函数替代。差异主要体现在 HIP 特有的半精度头文件、投票原语和纹理类型。

## 核心函数/宏定义

### 平台标识宏
- **`__KERNEL_GPU__`**: 标记当前为 GPU 内核编译
- **`__KERNEL_HIP__`**: 标记当前为 HIP 平台（对应 CUDA 的 `__KERNEL_CUDA__`）
- **`CCL_NAMESPACE_BEGIN` / `CCL_NAMESPACE_END`**: 定义为空

### 设备函数限定符
与 CUDA 版本完全一致的 `ccl_*` 宏映射：`ccl_device` -> `__device__ __inline__`、`ccl_device_forceinline` -> `__device__ __forceinline__` 等。

### `kernel_assert(cond)`
定义为空操作，HIP 设备端不支持断言。

### GPU 线程索引宏
与 CUDA 语法一致（HIP 兼容 CUDA 的线程模型）：`threadIdx.x`、`blockDim.x`、`blockIdx.x`、`gridDim.x`。

### 线程掩码差异
```c
#define ccl_gpu_thread_mask(thread_warp) uint64_t((1ull << thread_warp) - 1)
```
HIP 使用 64 位掩码（AMD GPU wavefront 宽度为 64），而 CUDA 使用 32 位掩码右移方式。

### GPU 同步差异
- **`ccl_gpu_ballot(predicate)`**: HIP 使用 `__ballot(predicate)`（无需显式同步掩码），而 CUDA 使用 `__ballot_sync(0xFFFFFFFF, predicate)`

### 纹理对象
- **`ccl_gpu_tex_object_2D`**: 类型为 `hipTextureObject_t`（而非 CUDA 的 `CUtexObject`）
- 读取函数同样使用 `tex2D<T>` 模板调用

### 半精度浮点
HIP 不需要手动定义半精度转换函数，而是通过系统头文件：
- `hip/hip_fp16.h` — HIP 半精度支持
- `hip/hip_runtime.h` — HIP 运行时

### MSVC 兼容
```c
#ifdef _MSC_VER
#  include <immintrin.h>
#endif
```
在 Windows MSVC 编译器下额外包含 SSE/AVX 内联函数头文件。

### 快速数学函数
与 CUDA 版本相同的替代宏：`cosf` -> `__cosf`、`sinf` -> `__sinf` 等。

### RTC 编译兼容
`__HIPCC_RTC__` 模式下手动定义 `uint32_t` 和 `uint64_t`（对应 CUDA 的 `__CUDACC_RTC__`）。

## 依赖关系

- **内部头文件**:
  - `hip/hip_fp16.h` (HIP 半精度，条件编译)
  - `hip/hip_runtime.h` (HIP 运行时，条件编译)
  - `util/half.h` (Cycles 半精度工具)
  - `util/types.h` (Cycles 类型系统)
- **被引用**:
  - `src/kernel/device/hip/kernel.cpp`
  - `src/kernel/device/hiprt/kernel.cpp`

## 实现细节 / 关键算法

### CUDA 与 HIP 兼容层差异总结
| 特性 | CUDA | HIP |
|------|------|-----|
| 平台宏 | `__KERNEL_CUDA__` | `__KERNEL_HIP__` |
| 线程掩码 | 32 位右移 | 64 位位移 |
| ballot | `__ballot_sync(0xFFFFFFFF, pred)` | `__ballot(pred)` |
| 纹理类型 | `CUtexObject` | `hipTextureObject_t` |
| 半精度 | PTX 内联汇编 | 系统头文件 |
| RTC 宏 | `__CUDACC_RTC__` | `__HIPCC_RTC__` |
| 浮点常量 | `CYCLES_CUBIN_CC` | `CYCLES_HIPBIN_CC` |

这些差异反映了 AMD 和 NVIDIA GPU 硬件架构的不同：AMD 的 wavefront 为 64 宽度（因此使用 64 位掩码），而 NVIDIA 的 warp 为 32 宽度。

## 关联文件

- `src/kernel/device/cuda/compat.h` — CUDA 兼容层（结构相同，细节不同）
- `src/kernel/device/cpu/compat.h` — CPU 兼容层（`kernel_assert` 为真实断言）
- `src/kernel/device/hip/config.h` — HIP 内核启动配置
- `src/kernel/device/hip/globals.h` — HIP 全局数据
