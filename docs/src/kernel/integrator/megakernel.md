# megakernel.h - 超级内核调度循环

## 概述

本文件实现了 Cycles 的超级内核（Megakernel）模式，将所有路径追踪内核整合到一个循环中执行。与波前（Wavefront）路径追踪模式（每个内核独立调度）不同，超级内核模式在单个函数调用中完成一条路径的全部计算。该模式主要用于 CPU 后端，其优势在于避免了内核调度的开销，而 GPU 端通常使用波前模式以获得更好的 SIMT 利用率。

## 核心函数

### integrator_megakernel()

- **签名**: `ccl_device void integrator_megakernel(KernelGlobals kg, IntegratorState state, ccl_global float *ccl_restrict render_buffer)`
- **功能**: 超级内核的主循环。通过检查积分器状态中的 `queued_kernel` 字段确定下一个要执行的内核，循环直到所有队列为空。执行优先级如下：
  1. **阴影路径**（最高优先级）：处理 `shadow` 状态中排队的 `INTERSECT_SHADOW` 或 `SHADE_SHADOW` 内核
  2. **AO 路径**：处理 `ao` 状态中排队的阴影内核
  3. **主路径**：处理主路径状态中排队的各种内核
- **支持的内核**:
  - 交点计算：`INTERSECT_CLOSEST`、`INTERSECT_SUBSURFACE`、`INTERSECT_VOLUME_STACK`、`INTERSECT_DEDICATED_LIGHT`
  - 着色：`SHADE_BACKGROUND`、`SHADE_SURFACE`、`SHADE_SURFACE_RAYTRACE`、`SHADE_SURFACE_MNEE`、`SHADE_VOLUME`、`SHADE_VOLUME_RAY_MARCHING`、`SHADE_LIGHT`、`SHADE_DEDICATED_LIGHT`
  - 阴影：`INTERSECT_SHADOW`、`SHADE_SHADOW`

## 依赖关系

- **内部头文件**:
  - `kernel/integrator/intersect_closest.h` - 最近交点内核
  - `kernel/integrator/intersect_dedicated_light.h` - 阴影链接专用光源内核
  - `kernel/integrator/intersect_shadow.h` - 阴影射线交点内核
  - `kernel/integrator/intersect_subsurface.h` - 次表面散射内核
  - `kernel/integrator/intersect_volume_stack.h` - 体积栈内核
  - `kernel/integrator/shade_background.h` - 背景着色内核
  - `kernel/integrator/shade_dedicated_light.h` - 专用光源着色内核
  - `kernel/integrator/shade_light.h` - 光源着色内核
  - `kernel/integrator/shade_shadow.h` - 阴影着色内核
  - `kernel/integrator/shade_surface.h` - 表面着色内核
  - `kernel/integrator/shade_volume.h` - 体积着色内核
- **被引用**:
  - `kernel/device/cpu/kernel_arch_impl.h` - CPU 架构实现（超级内核的唯一调用点）

## 实现细节 / 关键算法

1. **优先级调度**: 阴影路径优先于主路径处理，因为着色内核可能会创建新的阴影射线。优先处理已有阴影射线可以减少内存中同时活跃的路径状态数量。
2. **AO 路径**: 环境光遮蔽（AO）使用独立的阴影状态（`state->ao`），与常规阴影路径（`state->shadow`）分开处理。
3. **循环终止**: 当所有三个队列（shadow、ao、main）的 `queued_kernel` 均为 0 时，循环退出，表示该条路径的追踪全部完成。
4. **状态机模式**: 每个内核执行后会更新 `queued_kernel` 字段以指定下一步操作。这形成了一个隐式的状态机，超级内核循环是其调度器。
5. **GPU 不可用**: 此文件仅供 CPU 使用。GPU 通过波前模式分别调度每个内核，由主机端的工作队列协调。

## 关联文件

- `kernel/device/cpu/kernel_arch_impl.h` - CPU 端的超级内核调用入口
- `kernel/device/gpu/kernel.h` - GPU 端的波前内核调度（对比参考）
- 所有 `kernel/integrator/intersect_*.h` 和 `kernel/integrator/shade_*.h` - 被本文件包含并调用
