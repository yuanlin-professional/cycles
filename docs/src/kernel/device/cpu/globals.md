# globals.h + globals.cpp - CPU 设备内核全局数据结构定义与初始化

## 概述

本文件定义了 Cycles 渲染器 CPU 后端的全局数据结构体系。`globals.h` 声明了 `KernelGlobalsCPU`（共享全局数据）和 `ThreadKernelGlobalsCPU`（线程私有全局数据）两级结构，以及一系列数据访问抽象宏。`globals.cpp` 提供了线程全局数据的构造函数实现，包括 OSL 和路径引导的初始化逻辑以及性能分析器的注册与注销。

## 核心函数/宏定义

### 数据结构

#### `kernel_array<T>`
带边界检查的内核数据数组模板。`fetch(index)` 方法在调试模式下验证索引有效性，`data` 指针指向实际数据，`width` 记录数组大小。

#### `KernelGlobalsCPU`
所有线程共享的常量全局数据结构：
- 通过 `#include "kernel/data_arrays.h"` 宏展开生成所有 `kernel_array<type> name` 成员（如纹理信息、对象数据、曲线段数据等）
- `KernelData data` — 核心内核常量数据
- `ProfilingState profiler` — 性能分析状态

#### `ThreadKernelGlobalsCPU`
继承自 `KernelGlobalsCPU` 的线程私有数据：
- 构造函数从共享全局数据拷贝，避免指针间接寻址
- `OSLThreadData osl` — OSL 着色语言的线程数据（条件编译 `__OSL__`）
- 路径引导数据（条件编译 `__PATH_GUIDING__`）：
  - `opgl_sample_data_storage` — 共享采样数据存储指针
  - `opgl_guiding_field` — 共享引导场指针
  - `opgl_path_segment_storage` — 线程私有路径段存储
  - `opgl_surface_sampling_distribution` — 表面采样分布
  - `opgl_volume_sampling_distribution` — 体积采样分布

#### `KernelGlobals` 类型别名
```c
using KernelGlobals = const ThreadKernelGlobalsCPU *;
```
内核中传递的全局数据以指针形式存在，统一所有内核函数的参数接口。

### 数据访问宏
- **`kernel_data_fetch(name, index)`**: 从命名数组获取指定索引的元素（带边界检查）
- **`kernel_data_array(name)`**: 获取命名数组的原始数据指针
- **`kernel_data`**: 直接访问 `KernelData` 常量数据
- **`guiding_guiding_field`** / **`guiding_ssd`** / **`guiding_vsd`**: 路径引导的便捷访问宏

### 成员函数（globals.cpp）
- **`ThreadKernelGlobalsCPU::ThreadKernelGlobalsCPU()`**: 构造函数，拷贝共享全局数据，初始化 OSL 线程数据和路径引导存储
- **`ThreadKernelGlobalsCPU::start_profiling()`**: 向 CPU 分析器注册当前线程的分析状态
- **`ThreadKernelGlobalsCPU::stop_profiling()`**: 从 CPU 分析器注销当前线程的分析状态

## 依赖关系

- **内部头文件**:
  - `kernel/types.h` (内核类型定义)
  - `kernel/util/profiler.h` (性能分析器)
  - `kernel/osl/globals.h` (OSL 全局数据，条件编译)
  - `kernel/data_arrays.h` (数据数组宏展开)
  - `util/guiding.h` (路径引导库)
  - `util/texture.h` (纹理类型)
  - `util/unique_ptr.h` (智能指针)
  - `util/profiling.h` (分析工具，globals.cpp)
- **被引用**:
  - `src/kernel/globals.h`
  - `src/kernel/device/cpu/bvh.h`
  - `src/kernel/device/cpu/image.h`
  - `src/kernel/device/cpu/kernel.cpp`
  - `src/kernel/device/cpu/kernel_avx2.cpp`
  - `src/integrator/shader_eval.cpp`
  - `src/integrator/path_trace_work_cpu.h`

## 实现细节 / 关键算法

### 线程数据拷贝策略
为避免多线程访问共享数据时的指针间接寻址开销，`ThreadKernelGlobalsCPU` 直接从 `KernelGlobalsCPU` 拷贝构造。注释中提到这可能增加缓存压力，替代方案是传递线程索引并使用全局变量（更接近 GPU 模型），但需要互斥锁。

### 拷贝/移动语义
- 禁止拷贝构造和拷贝赋值（`= delete`）
- 允许移动构造（`= default`）
- 禁止移动赋值（`= delete`）

## 关联文件

- `src/kernel/device/cuda/globals.h` — CUDA 设备的全局数据，使用 `__constant__` 内存而非拷贝策略
- `src/kernel/device/hip/globals.h` — HIP 设备的全局数据，结构与 CUDA 类似
- `src/kernel/data_arrays.h` — 通过宏展开定义所有内核数据数组
- `src/kernel/device/cpu/kernel.cpp` — 使用本文件定义的结构进行内存拷贝
