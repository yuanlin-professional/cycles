# globals.h - Metal 设备全局数据结构与数据访问宏

## 概述

本文件定义了 Metal 后端的全局内核数据结构 `KernelParamsMetal`、空的 `KernelGlobalsGPU` 占位结构体，以及用于在内核代码中访问场景数据的抽象宏。它是 Metal 设备侧数据绑定的核心，将所有全局/常量数据数组和积分器状态组织到一个统一的参数结构体中。

## 核心函数/宏定义

### KernelParamsMetal 结构体

```cpp
struct KernelParamsMetal {
    #define KERNEL_DATA_ARRAY(type, name) ccl_global const type *name;
    #include "kernel/data_arrays.h"
    const IntegratorStateGPU integrator_state;
    const KernelData data;
};
```

通过 X 宏模式从 `data_arrays.h` 自动生成所有数据数组的指针成员（如顶点数据、纹理数据、BVH 节点等），加上积分器状态和内核常量数据。该结构体通过 Metal 的 `constant` 缓冲区绑定传递给内核。

### KernelGlobalsGPU 结构体

```cpp
struct KernelGlobalsGPU {
    int unused[1];
};
```

Metal 后端不使用传统的全局变量结构体（数据通过 `KernelParamsMetal` 传递），因此此结构体仅作占位用途。

### KernelGlobals 类型别名

```cpp
using KernelGlobals = const ccl_global KernelGlobalsGPU *ccl_restrict;
```

定义 `KernelGlobals` 为指向 `KernelGlobalsGPU` 的设备全局只读指针。在 Metal 后端中此指针通常为 `nullptr`，实际数据访问通过下方的宏进行。

### 数据访问宏

| 宏 | 展开 | 说明 |
|----|------|------|
| `kernel_data` | `launch_params_metal.data` | 访问 `KernelData` 常量数据 |
| `kernel_data_fetch(name, index)` | `launch_params_metal.name[index]` | 按索引访问数据数组元素 |
| `kernel_data_array(name)` | `launch_params_metal.name` | 获取数据数组指针 |
| `kernel_integrator_state` | `launch_params_metal.integrator_state` | 访问积分器状态 |

这些宏使得 GPU 通用内核代码无需感知 Metal 的具体数据绑定方式。

## 依赖关系

- **内部头文件**:
  - `kernel/types.h` — 内核基础类型定义
  - `kernel/integrator/state.h` — 积分器状态结构定义
  - `kernel/util/profiler.h` — 性能分析工具
  - `util/color.h` — 颜色工具函数
  - `util/texture.h` — 纹理工具定义
  - `kernel/data_arrays.h` — 通过 X 宏生成数据数组成员
- **被引用**:
  - `kernel/device/metal/kernel.metal` — Metal 内核入口文件
  - `device/metal/queue.h` — Metal 命令队列管理

## 实现细节 / 关键算法

### 与 MetalKernelContext 的关系

`KernelParamsMetal` 作为 `MetalKernelContext` 的成员（`launch_params_metal`），在内核入口点通过 Metal 缓冲区绑定传入。上述数据访问宏假设在 `MetalKernelContext` 的成员函数作用域内使用，因此可以直接通过 `this->launch_params_metal` 解析。

注意：`context_end.h` 会重新定义 `kernel_integrator_state` 为 `context.launch_params_metal.integrator_state`，以适配内核入口点 `run` 方法中的上下文变量命名。

## 关联文件

- `kernel/device/metal/compat.h` — Metal 兼容层，定义地址空间限定符
- `kernel/device/metal/context_begin.h` — MetalKernelContext 类定义
- `kernel/device/metal/context_end.h` — 宏重定向
- `kernel/device/metal/kernel.metal` — Metal 内核入口
- `kernel/data_arrays.h` — 数据数组定义模板
