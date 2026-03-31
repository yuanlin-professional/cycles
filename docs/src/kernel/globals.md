# globals.h - 内核全局状态访问入口

## 概述

`globals.h` 是一个轻量级的桥接头文件，用于在内核代码中统一引入 `KernelGlobals` 类型定义。它根据编译目标平台（CPU 或 GPU）选择性地包含对应的设备端实现头文件。对于 GPU 设备，`KernelGlobals` 在各自的设备头文件中预先定义；对于 CPU 设备，本文件引入 `kernel/device/cpu/compat.h` 和 `kernel/device/cpu/globals.h`。`KernelGlobals` 是贯穿整个内核代码的核心类型，承载了渲染所需的全部运行时数据访问接口。

## 类与结构体

本文件自身不定义任何结构体，但通过包含的头文件引入了 `KernelGlobals` 类型。在 CPU 端，`KernelGlobals` 通常包含：
- 指向所有 `data_arrays.h` 中声明的全局数据数组的指针
- `KernelData` 常量数据的引用
- OSL 着色系统全局状态（如启用）
- 路径引导相关状态（如启用）

## 枚举与常量

无。

## 核心函数

无函数定义。

## 依赖关系

- **内部头文件**:
  - `kernel/device/cpu/compat.h` — CPU 端兼容性定义（仅非 GPU 编译时）
  - `kernel/device/cpu/globals.h` — CPU 端 `KernelGlobals` 定义（仅非 GPU 编译时）
- **被引用**: 被内核中几乎所有需要访问全局数据的文件引用，共计约 51 个文件，包括但不限于：
  - SVM 着色器节点: `kernel/svm/svm.h`, `kernel/svm/image.h`, `kernel/svm/ao.h` 等
  - 几何处理: `kernel/geom/triangle.h`, `kernel/geom/curve.h`, `kernel/geom/object.h`, `kernel/geom/volume.h` 等
  - 光源采样: `kernel/light/light.h`, `kernel/light/background.h`, `kernel/light/distribution.h` 等
  - 积分器: `kernel/integrator/state_flow.h`, `kernel/integrator/guiding.h`, `kernel/integrator/intersect_shadow.h` 等
  - 胶片/成像: `kernel/film/write.h`, `kernel/film/cryptomatte_passes.h`
  - BVH 遍历: `kernel/bvh/nodes.h`, `kernel/bvh/util.h`
  - 摄像机: `kernel/camera/camera.h`
  - 烘焙: `kernel/bake/bake.h`
  - 设备端实现: `device/cpu/device_impl.cpp`, `device/cpu/device_impl.h`

## 实现细节 / 关键算法

本文件的设计理念是**平台抽象**。内核代码只需 `#include "kernel/globals.h"` 即可获得 `KernelGlobals` 类型，无需关心底层设备是 CPU、CUDA、HIP、Metal 还是 OneAPI。条件编译机制如下：

```
#ifndef __KERNEL_GPU__
  -> 包含 CPU 端头文件
#endif
// GPU 端：KernelGlobals 已由各设备的编译环境预定义
```

注释中提到本文件的另一目的是"使 clangd 工具正常工作"——即为代码分析工具提供 CPU 环境下的类型定义，以便在编辑器中获得正确的代码补全和错误检查。

## 关联文件

- `kernel/device/cpu/globals.h` — CPU 端 `KernelGlobals` 实际定义
- `kernel/device/cpu/compat.h` — CPU 端类型兼容层
- `kernel/device/cuda/globals.h` — CUDA 设备端定义
- `kernel/device/hip/globals.h` — HIP 设备端定义
- `kernel/device/metal/globals.h` — Metal 设备端定义
- `kernel/device/oneapi/globals.h` — OneAPI 设备端定义
- `kernel/device/optix/globals.h` — OptiX 设备端定义
- `kernel/data_arrays.h` — 定义 `KernelGlobals` 中包含的数据数组列表
- `kernel/types.h` — 定义 `KernelData` 及相关类型
