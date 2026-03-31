# globals.h - HIP 设备内核全局数据结构定义

## 概述

本文件定义了 Cycles 渲染器 HIP（AMD GPU）后端的全局数据结构。其设计与 CUDA 版本的 `globals.h` 完全对称，使用 `__constant__` 内存存储全局参数 `KernelParamsHIP`，并通过空 `KernelGlobalsGPU` 占位结构和宏抽象层提供统一的数据访问接口。

## 核心函数/宏定义

### `KernelGlobalsGPU`
与 CUDA 版本相同的空占位结构体（`int unused[1]`），仅为保持内核函数签名一致。

### `KernelGlobals` 类型别名
```c
using KernelGlobals = const ccl_global KernelGlobalsGPU *ccl_restrict;
```
GPU 上为指向空结构的 const 指针。

### `KernelParamsHIP`
HIP 特有的参数结构体，结构与 `KernelParamsCUDA` 完全对称：
- **`KernelData data`**: 核心内核常量数据
- **数据数组指针**: 通过 `#include "kernel/data_arrays.h"` 展开的 `const type *name` 成员
- **`IntegratorStateGPU integrator_state`**: GPU 积分器状态

### `kernel_params`
```c
__constant__ KernelParamsHIP kernel_params;
```
在 `__constant__` 内存中声明的全局参数实例。

### 数据访问抽象宏
与 CUDA 版本完全相同：
- **`kernel_data`**: `kernel_params.data`
- **`kernel_data_fetch(name, index)`**: `kernel_params.name[(index)]`
- **`kernel_data_array(name)`**: `(kernel_params.name)`
- **`kernel_integrator_state`**: `kernel_params.integrator_state`

## 依赖关系

- **内部头文件**:
  - `kernel/types.h` (内核类型定义)
  - `kernel/integrator/state.h` (积分器状态)
  - `kernel/util/profiler.h` (分析器)
  - `util/color.h` (颜色工具)
  - `util/texture.h` (纹理类型)
- **被引用**:
  - `src/kernel/device/hip/kernel.cpp`
  - `src/device/hip/device_impl.cpp`（设备端将主机数据拷贝到 `__constant__` 内存）

## 实现细节 / 关键算法

### HIP __constant__ 内存
HIP 的 `__constant__` 内存与 CUDA 语义相同：
- 存储在设备全局内存中，但通过专用缓存加速访问
- 所有线程只读访问同一数据时效率最高
- 大小通常限制为 64KB

### 与 CUDA 版本的唯一差异
两个文件的结构和宏完全相同，唯一区别是参数结构体名称：
- CUDA: `KernelParamsCUDA`
- HIP: `KernelParamsHIP`

这种分离是为了避免同时编译多个后端时的类型名冲突。

## 关联文件

- `src/kernel/device/cuda/globals.h` — CUDA 版本（结构完全对称）
- `src/kernel/device/cpu/globals.h` — CPU 版本（使用线程拷贝策略）
- `src/kernel/data_arrays.h` — 数据数组列表的宏定义
- `src/device/hip/device_impl.cpp` — 主机端将数据拷贝到 `kernel_params`
