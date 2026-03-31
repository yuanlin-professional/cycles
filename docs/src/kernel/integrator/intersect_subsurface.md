# intersect_subsurface.h - 次表面散射交点计算

## 概述

本文件实现了次表面散射（Subsurface Scattering, SSS）的交点计算内核。次表面散射模拟光线进入半透明材质后在内部散射并从另一点射出的效果，常用于皮肤、蜡烛、大理石等材质的渲染。该内核是波前路径追踪架构中的一个专用内核，负责调用次表面散射采样并在完成后终止路径或继续追踪。

## 核心函数

### integrator_intersect_subsurface()

- **签名**: `ccl_device void integrator_intersect_subsurface(KernelGlobals kg, IntegratorState state)`
- **功能**: 次表面散射交点内核的主入口。在 `__SUBSURFACE__` 编译宏保护下，调用 `subsurface_scatter` 执行实际的次表面散射采样（包括随机游走或盘采样）。若 `subsurface_scatter` 返回 `true`，表示散射成功并且已调度了后续内核；若返回 `false`，则调用 `integrator_path_terminate` 终止路径。

## 依赖关系

- **内部头文件**:
  - `kernel/integrator/subsurface.h` - 次表面散射核心实现（`subsurface_scatter` 函数）
- **被引用**:
  - `kernel/integrator/megakernel.h` - 超级内核循环
  - `kernel/device/optix/kernel.cu` - OptiX 内核
  - `kernel/device/gpu/kernel.h` - GPU 内核

## 实现细节

1. **轻量封装**: 该文件是一个非常简洁的内核入口，实际的次表面散射算法（随机游走、盘采样等）由 `kernel/integrator/subsurface.h` 及其子模块实现。
2. **条件编译**: 整个次表面散射功能受 `__SUBSURFACE__` 宏控制，在不支持 SSS 的构建配置中该内核不会执行任何操作。
3. **路径终止**: 若散射失败（例如射线未找到出射点），路径将以 `render_buffer = nullptr` 终止，不写入任何渲染结果。

## 关联文件

- `kernel/integrator/subsurface.h` - 次表面散射调度逻辑
- `kernel/integrator/subsurface_disk.h` - 盘采样次表面散射实现
- `kernel/integrator/subsurface_random_walk.h` - 随机游走次表面散射实现
- `kernel/integrator/intersect_volume_stack.h` - SSS 出射点的体积栈更新
