# kernel_osl.cu - OptiX 开放着色语言(OSL)完整内核编译入口

## 概述

本文件是启用开放着色语言（OSL）支持的 OptiX 内核编译入口。它是标准 OptiX 内核的超集，在包含着色器光线追踪内核（`kernel_shader_raytrace.cu`）的基础上，额外包含了所有着色阶段和着色器评估的 raygen 程序。这些程序在场景使用 OSL 着色器时被加载，提供完整的渲染管线支持。

## 核心函数/宏定义

### 编译标志

- **`#define WITH_OSL`** - 启用 OSL 支持，在编译时激活 OSL 相关的代码路径。

### 着色阶段 raygen 程序（8个）

- **`__raygen__kernel_optix_integrator_shade_background()`** - 背景着色。对未命中任何物体的光线计算背景/环境光照。
- **`__raygen__kernel_optix_integrator_shade_light()`** - 光源着色。计算直接光照贡献。
- **`__raygen__kernel_optix_integrator_shade_surface()`** - 表面着色。执行 BSDF 评估和光照积分。
- **`__raygen__kernel_optix_integrator_shade_volume()`** - 体积着色。处理体积散射和吸收。
- **`__raygen__kernel_optix_integrator_shade_volume_ray_marching()`** - 体积光线步进着色。使用光线步进算法处理异构体积。
- **`__raygen__kernel_optix_integrator_shade_shadow()`** - 阴影着色。处理透明阴影的衰减计算。
- **`__raygen__kernel_optix_integrator_shade_dedicated_light()`** - 专用光源着色。

### 着色器评估 raygen 程序（4个）

- **`__raygen__kernel_optix_shader_eval_displace()`** - 位移着色器评估。从 `KernelShaderEvalInput` 输入计算位移偏移。
- **`__raygen__kernel_optix_shader_eval_background()`** - 背景着色器评估。预计算背景贴图。
- **`__raygen__kernel_optix_shader_eval_curve_shadow_transparency()`** - 曲线阴影透明度评估。预烘焙曲线的阴影透明度值。
- **`__raygen__kernel_optix_shader_eval_volume_density()`** - 体积密度评估。预计算体积密度场。

## 依赖关系

- **内部头文件**:
  - `kernel/device/optix/kernel_shader_raytrace.cu` - 着色器光线追踪内核（包含了 `kernel.cu` 的全部内容）
  - `kernel/bake/bake.h` - 烘焙功能
  - `kernel/integrator/shade_background.h` - 背景着色
  - `kernel/integrator/shade_dedicated_light.h` - 专用光源着色
  - `kernel/integrator/shade_light.h` - 光源着色
  - `kernel/integrator/shade_shadow.h` - 阴影着色
  - `kernel/integrator/shade_volume.h` - 体积着色
  - `kernel/device/gpu/work_stealing.h` - GPU 工作窃取调度

- **被引用**: 本文件作为独立编译单元，由构建系统直接编译，不被其他源文件包含。

## 实现细节 / 关键算法

1. **分层包含结构**: 本文件通过 `#include "kernel_shader_raytrace.cu"` 间接包含了 `kernel.cu`，因此包含了完整的求交阶段和着色器光线追踪阶段的所有程序。在此基础上添加着色阶段和着色器评估程序，形成完整的 OSL 渲染管线。

2. **着色器评估的特殊参数传递**: 着色器评估程序（displace/background/curve_shadow_transparency/volume_density）复用 `kernel_params.path_index_array` 作为 `KernelShaderEvalInput` 输入指针，使用 `kernel_params.offset` 作为全局索引偏移。

3. **OSL 与 SVM 的编译分离**: `WITH_OSL` 宏影响着色器编译路径。OSL 内核单独编译为独立的 PTX/OptiX 模块，仅在场景使用 OSL 着色器时才被加载到 OptiX 管线中，避免非 OSL 场景的额外编译开销。

4. **体积光线步进**: `integrator_shade_volume_ray_marching` 是标准体积着色的变体，使用光线步进（ray marching）算法处理空间变化的体积属性，提供更精确但更慢的体积渲染。

## 关联文件

- `kernel/device/optix/kernel_shader_raytrace.cu` - 被本文件包含，提供求交和光线追踪着色程序
- `kernel/device/optix/kernel.cu` - 间接包含，提供基础求交程序
- `kernel/device/optix/kernel_osl_camera.cu` - OSL 相机初始化内核（独立编译）
- `kernel/integrator/shade_surface.h` - 表面着色实现（通过 `kernel_shader_raytrace.cu` 包含）
- `kernel/bake/bake.h` - 烘焙相关的着色器评估
- `device/optix/device_impl.cpp` - 创建 OptiX 管线时加载本编译单元生成的模块
