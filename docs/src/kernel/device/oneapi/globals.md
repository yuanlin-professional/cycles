# globals.h - oneAPI 设备全局数据结构与数据访问宏

## 概述

本文件定义了 oneAPI 后端的全局内核数据结构 `KernelGlobalsGPU`，以及用于在内核代码中访问场景数据的抽象宏。与 Metal 后端将数据存储在单独的 `KernelParams` 结构体中不同，oneAPI 将所有数据指针直接作为 `KernelGlobalsGPU` 的成员，因为 SYCL 不支持 CUDA 风格的 `__constant__` 全局变量。

## 核心函数/宏定义

### KernelGlobalsGPU 结构体

```cpp
struct KernelGlobalsGPU {
    #define KERNEL_DATA_ARRAY(type, name) const type *__##name = nullptr;
    #include "kernel/data_arrays.h"
    IntegratorStateGPU *integrator_state;
    const KernelData *__data;
    // 执行模式相关成员...
};
```

**数据成员**:
- 通过 X 宏从 `data_arrays.h` 自动生成所有数据数组指针（以 `__` 前缀命名，如 `__tri_verts`、`__objects` 等）
- `integrator_state` — 积分器状态 GPU 数据指针
- `__data` — `KernelData` 常量数据指针

**执行模式成员**:
- `WITH_ONEAPI_SYCL_HOST_TASK` 模式：模拟的 `nd_item` 值（`nd_item_local_id_0`、`nd_item_global_id_0` 等）
- 标准模式：`sycl::kernel_handler kernel_handler`（用于读取 SYCL 特化常量）

### 前向声明

```cpp
struct IntegratorStateGPU;
struct IntegratorQueueCounter;
```

### KernelGlobals 类型别名

```cpp
using KernelGlobals = ccl_global KernelGlobalsGPU *ccl_restrict;
```

定义 `KernelGlobals` 为指向 `KernelGlobalsGPU` 的受限指针。在 oneAPI 后端中，此指针指向通过 USM（统一共享内存）分配的设备内存。

### 数据访问宏

| 宏 | 展开 | 说明 |
|----|------|------|
| `kernel_data` | `(*(__data))` | 解引用 KernelData 指针 |
| `kernel_integrator_state` | `(*(integrator_state))` | 解引用积分器状态指针 |
| `kernel_data_fetch(name, index)` | `__##name[index]` | 按索引访问数据数组（使用 `__` 前缀成员） |
| `kernel_data_array(name)` | `__##name` | 获取数据数组指针 |

注意：`kernel_integrator_state` 会在 `context_end.h` 中被重新定义为 `(*(kg->integrator_state))`。

## 依赖关系

- **内部头文件**:
  - `kernel/types.h` — 内核基础类型定义
  - `kernel/integrator/state.h` — 积分器状态结构定义
  - `kernel/util/profiler.h` — 性能分析工具
  - `util/color.h` — 颜色工具函数
  - `util/texture.h` — 纹理工具定义
  - `kernel/data_arrays.h` — 通过 X 宏生成数据数组成员
- **被引用**:
  - `kernel/device/oneapi/kernel.cpp` — oneAPI 内核入口文件
  - `kernel/device/cpu/bvh.h` — CPU 后端的 Embree BVH 遍历（复用 oneAPI 全局结构）
  - `device/oneapi/device_impl.cpp` — 主机端设备实现

## 实现细节 / 关键算法

### 与 Metal 实现的对比

| 特性 | Metal (`globals.h`) | oneAPI (`globals.h`) |
|------|---------------------|----------------------|
| 数据存储 | `KernelParamsMetal` 独立结构体 | `KernelGlobalsGPU` 直接存储 |
| KernelData | 值成员 (`const KernelData data`) | 指针成员 (`const KernelData *__data`) |
| 积分器状态 | 值成员 | 指针成员 |
| KernelGlobals 角色 | 占位（unused） | 核心数据容器 |

oneAPI 使用指针而非值成员的原因是 SYCL 内核通过 USM 设备内存传递数据，主机端分配内存后将指针赋值给 `KernelGlobalsGPU` 的成员。

### ONEAPIKernelContext 继承

`ONEAPIKernelContext` 继承自 `KernelGlobalsGPU`，因此在上下文成员函数中，数据访问宏中的 `__##name` 可以通过 `this->__##name` 隐式解析。

## 关联文件

- `kernel/device/oneapi/compat.h` — oneAPI 兼容层
- `kernel/device/oneapi/context_begin.h` — ONEAPIKernelContext 类（继承自 KernelGlobalsGPU）
- `kernel/device/oneapi/context_end.h` — 宏重定向
- `kernel/device/oneapi/kernel.cpp` — oneAPI 内核入口
- `kernel/data_arrays.h` — 数据数组定义模板
