# globals.h - CUDA 设备内核全局数据结构定义

## 概述

本文件定义了 Cycles 渲染器 CUDA 后端的全局数据结构。与 CPU 版本不同，CUDA 使用 `__constant__` 内存存储全局参数（`KernelParamsCUDA`），并通过一个空的 `KernelGlobalsGPU` 占位结构和宏抽象层提供与 CPU 内核统一的数据访问接口。GPU 后端的 `KernelGlobals` 本质上是一个未使用的空指针，所有数据访问通过全局 `kernel_params` 完成。

## 核心函数/宏定义

### `KernelGlobalsGPU`
仅包含一个 `int unused[1]` 成员的空结构体。GPU 内核不需要通过指针传递全局数据（数据存储在 `__constant__` 内存中），但为保持与 CPU 内核统一的函数签名，仍然传递一个 `KernelGlobals` 参数，编译器会将其优化掉。

### `KernelGlobals` 类型别名
```c
using KernelGlobals = const ccl_global KernelGlobalsGPU *ccl_restrict;
```
GPU 上为指向空结构的 const 指针。

### `KernelParamsCUDA`
CUDA 特有的参数结构，存储在 `__constant__` 内存中：
- **`KernelData data`**: 核心内核常量数据（场景参数、积分器设置等）
- **数据数组指针**: 通过 `#include "kernel/data_arrays.h"` 宏展开，为每种数据类型生成 `const type *name` 指针成员（指向设备全局内存中的实际数据）
- **`IntegratorStateGPU integrator_state`**: GPU 积分器状态

### `kernel_params`
```c
__constant__ KernelParamsCUDA kernel_params;
```
在 `__constant__` 内存中声明的全局参数实例。`__constant__` 内存由 GPU 硬件缓存，适合存储所有线程只读的小型数据。

### 数据访问抽象宏
- **`kernel_data`**: 展开为 `kernel_params.data`
- **`kernel_data_fetch(name, index)`**: 展开为 `kernel_params.name[(index)]`，直接从全局内存数组索引
- **`kernel_data_array(name)`**: 展开为 `(kernel_params.name)`，获取数组指针
- **`kernel_integrator_state`**: 展开为 `kernel_params.integrator_state`

## 依赖关系

- **内部头文件**:
  - `kernel/types.h` (内核类型定义)
  - `kernel/integrator/state.h` (积分器状态)
  - `kernel/util/profiler.h` (分析器)
  - `util/color.h` (颜色工具)
  - `util/texture.h` (纹理类型)
- **被引用**:
  - `src/kernel/device/cuda/kernel.cu`
  - `src/device/cuda/device_impl.cpp`（设备端将主机数据拷贝到 `__constant__` 内存）

## 实现细节 / 关键算法

### CPU 与 GPU 全局数据的架构差异
- **CPU**: `KernelGlobals` 是指向 `ThreadKernelGlobalsCPU` 的指针，每个线程持有独立副本，所有数据通过指针解引用访问
- **CUDA GPU**: `KernelGlobals` 是未使用的空指针，数据访问通过 `__constant__` 内存中的 `kernel_params` 全局变量完成

这种设计利用了 CUDA `__constant__` 内存的硬件缓存特性，避免每个线程的间接寻址开销。数据数组指针存储在 `__constant__` 中，指向设备全局内存中的实际数据。

### 统一访问接口
尽管底层机制完全不同，`kernel_data_fetch`、`kernel_data` 等宏在 CPU 和 GPU 上提供相同的语义接口，使内核主体代码无需关心设备差异。

## 关联文件

- `src/kernel/device/cpu/globals.h` — CPU 版本，使用线程拷贝策略而非 `__constant__` 内存
- `src/kernel/device/hip/globals.h` — HIP 版本，结构与 CUDA 几乎相同（`KernelParamsHIP`）
- `src/kernel/data_arrays.h` — 数据数组列表的宏定义
- `src/device/cuda/device_impl.cpp` — 主机端通过 `cuMemcpyHtoD` 将数据拷贝到 `kernel_params`
