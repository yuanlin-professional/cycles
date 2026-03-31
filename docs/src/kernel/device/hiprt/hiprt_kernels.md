# hiprt_kernels.h - HIPRT 专用 GPU 内核入口点定义

## 概述

本文件定义了需要 HIPRT 硬件光线追踪支持的 GPU 内核入口点。这些内核是标准 GPU 内核的 HIPRT 特化版本，与普通 HIP 内核的区别在于：它们接受额外的 `hiprtGlobalStackBuffer` 参数用于 BVH 遍历，并在每个内核入口处通过 `HIPRT_INIT_KERNEL_GLOBAL()` 宏初始化共享内存栈。本文件受 `__HIPRT__` 宏保护。

## 核心函数/宏定义

### 求交内核（5个）

- **`integrator_intersect_closest`** - 最近交点求交内核，接收路径索引数组、渲染缓冲区、工作大小和全局栈缓冲区。调用积分器的 `integrator_intersect_closest` 函数。
- **`integrator_intersect_shadow`** - 阴影光线求交内核。
- **`integrator_intersect_subsurface`** - 子表面散射求交内核。
- **`integrator_intersect_volume_stack`** - 体积栈求交内核。
- **`integrator_intersect_dedicated_light`** - 专用光源求交内核。

### 着色内核（2个）

- **`integrator_shade_surface_raytrace`** - 表面光线追踪着色内核，用于需要在着色过程中发射额外光线的材质（如折射、SSS 等）。
- **`integrator_shade_surface_mnee`** - 流形下一事件估计（MNEE）着色内核，用于焦散光路的高效采样。

### 通用模式

所有内核遵循统一的模式：

1. 使用 `ccl_gpu_kernel_threads(GPU_HIPRT_KERNEL_BLOCK_NUM_THREADS)` 设置线程块大小
2. 从 `ccl_gpu_global_id_x()` 获取全局线程索引
3. 进行工作范围检查 `global_index < work_size`
4. 调用 `HIPRT_INIT_KERNEL_GLOBAL()` 初始化共享内存栈和全局状态
5. 从路径索引数组解析状态索引
6. 调用对应的积分器函数

## 依赖关系

- **内部头文件**: 无显式 `#include`（通过编译环境和 `kernel.cpp` 间接获得所有依赖）

- **被引用**:
  - `kernel/device/gpu/kernel.h` - GPU 通用内核头文件条件包含本文件

## 实现细节 / 关键算法

1. **HIPRT 内核与标准 HIP 内核的区别**: 标准 HIP 内核使用软件 BVH 遍历，不需要全局栈缓冲区参数。HIPRT 内核通过额外的 `hiprtGlobalStackBuffer stack_buffer` 参数接收主机端分配的遍历栈内存，并在内核入口处初始化共享内存栈。

2. **路径索引间接寻址**: `path_index_array` 可为 `nullptr`，此时使用 `global_index` 直接作为路径状态索引。非空时通过间接寻址获取实际的路径状态索引，支持路径紧缩等优化。

3. **线程块大小**: 使用 `GPU_HIPRT_KERNEL_BLOCK_NUM_THREADS` 而非通用的 `GPU_KERNEL_BLOCK_NUM_THREADS`，因为 HIPRT 内核的共享内存使用量不同，需要不同的线程块大小配置。

## 关联文件

- `kernel/device/hiprt/globals.h` - 提供 `HIPRT_INIT_KERNEL_GLOBAL` 宏和 `KernelGlobalsGPU` 定义
- `kernel/device/gpu/kernel.h` - 通用 GPU 内核文件，条件引入本文件
- `kernel/integrator/intersect_closest.h` - 最近求交积分器实现
- `kernel/integrator/intersect_shadow.h` - 阴影求交积分器实现
- `kernel/integrator/intersect_subsurface.h` - 子表面散射求交实现
- `kernel/integrator/intersect_volume_stack.h` - 体积栈求交实现
- `kernel/integrator/intersect_dedicated_light.h` - 专用光源求交实现
- `kernel/integrator/shade_surface.h` - 表面着色实现
