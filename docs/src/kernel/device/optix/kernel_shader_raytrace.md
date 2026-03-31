# kernel_shader_raytrace.cu - OptiX 着色器光线追踪内核

## 概述

本文件定义了需要在着色过程中发射额外光线的 OptiX 内核程序。这些程序的编译时间显著长于普通着色内核，因此被独立分离出来，仅在场景实际需要时才加载。包含表面光线追踪着色和流形下一事件估计（MNEE）两个 raygen 程序。

## 核心函数/宏定义

### raygen 程序

- **`__raygen__kernel_optix_integrator_shade_surface_raytrace()`** - 表面光线追踪着色程序。在表面着色过程中需要发射额外光线（如用于 SSS 的探测光线、折射光线等）。调用 `integrator_shade_surface_raytrace` 积分器函数。

- **`__raygen__kernel_optix_integrator_shade_surface_mnee()`** - 流形下一事件估计（Manifold Next Event Estimation）着色程序。MNEE 是一种针对焦散光路（通过光滑折射/反射表面的间接光照）的高效采样算法。调用 `integrator_shade_surface_mnee` 积分器函数。

### 头文件组织

本文件通过 `#include "kernel.cu"` 包含完整的 OptiX 主内核，然后额外包含 `kernel/integrator/shade_surface.h` 提供表面着色实现。

## 依赖关系

- **内部头文件**:
  - `kernel/device/optix/kernel.cu` - OptiX 主内核（提供所有求交程序和基础设施）
  - `kernel/integrator/shade_surface.h` - 表面着色积分器实现

- **被引用**:
  - `kernel/device/optix/kernel_osl.cu` - OSL 内核通过 `#include` 包含本文件

## 实现细节 / 关键算法

1. **编译时间优化**: 注释明确指出本文件"编译时间远长于普通内核"。`shade_surface.h` 中包含了完整的 BSDF 评估、光照采样和光线追踪逻辑，代码量巨大。将其从主内核分离出来，使得不需要光线追踪着色的简单场景可以更快地完成编译。

2. **按需加载**: 此内核模块仅在场景包含需要光线追踪着色的材质时才被加载到 OptiX 管线中。主机端代码（`device_impl.cpp`）会根据场景特性决定是否加载此模块。

3. **MNEE 算法**: 流形下一事件估计是一种先进的光传输算法，用于高效渲染通过多层光滑界面（如玻璃）的焦散效果。它通过牛顿迭代在光滑流形上寻找连接光源和相机的特殊路径，显著减少焦散场景的噪声。

4. **包含链**: 本文件形成了一个三级包含链：`kernel_osl.cu` -> `kernel_shader_raytrace.cu` -> `kernel.cu`。每一级添加更多的渲染功能，最终形成完整的 OSL 渲染管线。

## 关联文件

- `kernel/device/optix/kernel.cu` - 被本文件包含的主内核
- `kernel/device/optix/kernel_osl.cu` - 包含本文件的 OSL 完整内核
- `kernel/integrator/shade_surface.h` - 表面着色核心实现
- `kernel/integrator/mnee.h` - MNEE 算法实现（通过 `shade_surface.h` 间接包含）
- `device/optix/device_impl.cpp` - 管线模块管理和按需加载逻辑
