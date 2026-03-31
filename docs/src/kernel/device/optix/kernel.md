# kernel.cu - OptiX 主内核编译入口与光线生成程序

## 概述

本文件是 OptiX 后端的主编译单元，包含所有积分器求交阶段的光线生成（raygen）程序。这些 raygen 程序是 OptiX 管线的入口点，由 `optixLaunch` 启动。每个 raygen 程序对应积分器的一个求交阶段，从 `kernel_params` 获取路径索引，调用相应的积分器函数完成光线遍历和求交计算。

## 核心函数/宏定义

### 光线生成程序（5个求交阶段）

- **`__raygen__kernel_optix_integrator_intersect_closest()`** - 最近交点求交。从 `optixGetLaunchIndex().x` 获取全局索引，通过路径索引数组解析路径状态，调用 `integrator_intersect_closest`。

- **`__raygen__kernel_optix_integrator_intersect_shadow()`** - 阴影光线求交，调用 `integrator_intersect_shadow`。

- **`__raygen__kernel_optix_integrator_intersect_subsurface()`** - 子表面散射求交，调用 `integrator_intersect_subsurface`。

- **`__raygen__kernel_optix_integrator_intersect_volume_stack()`** - 体积栈求交，调用 `integrator_intersect_volume_stack`。

- **`__raygen__kernel_optix_integrator_intersect_dedicated_light()`** - 专用光源求交，调用 `integrator_intersect_dedicated_light`。

### 头文件组织

本文件按顺序包含以下关键头文件：

1. `kernel/device/optix/compat.h` - OptiX 兼容层
2. `kernel/device/optix/globals.h` - 全局参数
3. `kernel/device/gpu/image.h` - GPU 纹理采样
4. `kernel/tables.h` - 内核查找表
5. `kernel/integrator/state.h` / `state_flow.h` / `state_util.h` - 积分器状态管理
6. `kernel/integrator/intersect_*.h` - 各求交阶段实现

## 依赖关系

- **内部头文件**:
  - `kernel/device/optix/compat.h` - 平台兼容层
  - `kernel/device/optix/globals.h` - 全局参数定义
  - `kernel/device/gpu/image.h` - GPU 图像处理
  - `kernel/tables.h` - 查找表
  - `kernel/integrator/state.h` - 积分器状态
  - `kernel/integrator/state_flow.h` - 状态流转
  - `kernel/integrator/state_util.h` - 状态工具
  - `kernel/integrator/intersect_closest.h` - 最近求交
  - `kernel/integrator/intersect_shadow.h` - 阴影求交
  - `kernel/integrator/intersect_subsurface.h` - 子表面散射求交
  - `kernel/integrator/intersect_volume_stack.h` - 体积栈求交
  - `kernel/integrator/intersect_dedicated_light.h` - 专用光源求交

- **被引用**:
  - `kernel/device/optix/kernel_shader_raytrace.cu` - 着色器光线追踪内核通过 `#include` 包含本文件

## 实现细节 / 关键算法

1. **KernelGlobals 为 nullptr**: 所有积分器函数的第一个参数传递 `nullptr` 作为 `KernelGlobals`。OptiX 内核不需要显式的全局状态指针，所有数据通过 `__constant__` 内存中的 `kernel_params` 访问。

2. **路径索引间接寻址**: 与 HIPRT 内核类似，`kernel_params.path_index_array` 为 `nullptr` 时直接使用 `global_index`，否则通过间接寻址获取实际路径索引。

3. **OptiX 启动维度**: 所有 raygen 程序使用一维启动（`optixGetLaunchIndex().x`），工作大小由主机端 `optixLaunch` 的启动参数控制。

4. **编译单元复用**: `kernel_shader_raytrace.cu` 通过 `#include "kernel.cu"` 复用本文件的全部内容，避免代码重复。这意味着着色器光线追踪内核包含了所有求交阶段的程序。

## 关联文件

- `kernel/device/optix/compat.h` - 兼容层
- `kernel/device/optix/globals.h` - 全局参数
- `kernel/device/optix/bvh.h` - BVH 求交实现（间接包含）
- `kernel/device/optix/kernel_shader_raytrace.cu` - 包含本文件的着色器光线追踪内核
- `kernel/device/optix/kernel_osl.cu` - OSL 内核（包含 `kernel_shader_raytrace.cu`，间接包含本文件）
- `device/optix/device_impl.cpp` - 主机端设备实现，创建 OptiX 管线并启动这些 raygen 程序
