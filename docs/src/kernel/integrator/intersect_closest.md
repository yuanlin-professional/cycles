# intersect_closest.h - 最近交点计算与路径调度

## 概述

本文件是 Cycles 波前路径追踪架构中的核心交点计算内核。它负责对积分器状态中的射线执行场景层次包围体(BVH)遍历，找到最近交点，并根据交点类型（表面、光源、背景或体积）调度后续的着色内核。该文件还包含路径终止判断（俄罗斯轮盘赌、AO 弹射近似）、Shadow Catcher 路径分裂以及流形下一事件估计(MNEE)防火花逻辑。

## 核心函数

### integrator_intersect_skip_lights()

- **签名**: `ccl_device_forceinline bool integrator_intersect_skip_lights(KernelGlobals kg, IntegratorState state)`
- **功能**: 判断在烘焙模式下是否跳过光源评估。当直接光照被禁用且处于第一次弹射时返回 `true`，以保持 MIS 一致性。

### integrator_intersect_terminate()

- **签名**: `ccl_device_forceinline bool integrator_intersect_terminate(KernelGlobals kg, IntegratorState state, const int shader_flags)`
- **功能**: 在交点处执行路径终止判断。包含两个机制：
  1. **AO 弹射终止**: 当路径弹射次数超过 AO 弹射限制时，若着色器有透明或自发光则标记延迟终止，否则立即终止。
  2. **俄罗斯轮盘赌**: 基于路径通量计算继续概率，随机决定是否终止路径。对有自发光着色器的表面，标记 `PATH_RAY_TERMINATE_ON_NEXT_SURFACE` 以在评估完自发光后终止。

### integrator_split_shadow_catcher()

- **签名**: `ccl_device_forceinline void integrator_split_shadow_catcher(KernelGlobals kg, IntegratorState state, const ccl_private Intersection *ccl_restrict isect, ccl_global float *ccl_restrict render_buffer)`
- **功能**: 当射线击中 Shadow Catcher 对象时分裂路径。创建一个新的积分器状态用于追踪 Shadow Catcher 遮罩通道，原路径继续贡献合成通道。分裂后的路径可能需要重新初始化体积栈或渲染背景通道。

### integrator_intersect_next_kernel\<current_kernel\>()

- **签名**: `template<DeviceKernel current_kernel> ccl_device_forceinline void integrator_intersect_next_kernel(KernelGlobals kg, IntegratorState state, const ccl_private Intersection *ccl_restrict isect, ccl_global float *ccl_restrict render_buffer, const bool hit)`
- **功能**: 根据交点结果调度下一个内核。逻辑如下：
  - **体积内部**: 若体积栈非空，调度 `SHADE_VOLUME` 或 `SHADE_VOLUME_RAY_MARCHING`
  - **击中表面**: 根据着色器标志选择 `SHADE_SURFACE`、`SHADE_SURFACE_RAYTRACE` 或 `SHADE_SURFACE_MNEE`
  - **击中光源**: 调度 `SHADE_LIGHT`
  - **未击中**: 调度 `SHADE_BACKGROUND`
- **模板参数**: `current_kernel` 作为模板参数以优化 CUDA 原子操作性能。

### integrator_intersect_next_kernel_after_volume\<current_kernel\>()

- **签名**: `template<DeviceKernel current_kernel> ccl_device_forceinline void integrator_intersect_next_kernel_after_volume(KernelGlobals kg, IntegratorState state, const ccl_private Intersection *ccl_restrict isect, ccl_global float *ccl_restrict render_buffer)`
- **功能**: 体积着色完成后调度下一内核。逻辑与 `integrator_intersect_next_kernel` 类似，但跳过体积着色和终止测试（已在体积着色内核中完成）。

### integrator_intersect_closest()

- **签名**: `ccl_device void integrator_intersect_closest(KernelGlobals kg, IntegratorState state, ccl_global float *ccl_restrict render_buffer)`
- **功能**: 最近交点计算的主入口函数。执行流程：
  1. 从积分器状态读取射线
  2. 处理 AO 弹射近似（缩短射线最大距离）
  3. 调用 `scene_intersect` 执行 BVH 遍历
  4. 执行 MNEE 防火花路径剔除逻辑
  5. 执行光源 MIS 交点查询
  6. 将交点写入积分器状态
  7. 调用 `integrator_intersect_next_kernel` 调度后续内核

### integrator_intersect_next_kernel_after_shadow_catcher_volume\<current_kernel\>() / integrator_intersect_next_kernel_after_shadow_catcher_background\<current_kernel\>()

- **功能**: Shadow Catcher 分裂路径在完成体积栈更新或背景着色后，调度后续表面着色内核。

## 依赖关系

- **内部头文件**:
  - `kernel/film/light_passes.h` - 光线通道写入
  - `kernel/integrator/guiding.h` - 路径引导记录
  - `kernel/integrator/path_state.h` - 路径状态和随机数
  - `kernel/integrator/shadow_catcher.h` - Shadow Catcher 工具
  - `kernel/light/light.h` - 光源交点计算
  - `kernel/bvh/bvh.h` - 层次包围体(BVH)遍历
- **被引用**:
  - `kernel/integrator/init_from_bake.h` - 烘焙初始化
  - `kernel/integrator/intersect_volume_stack.h` - 体积栈初始化
  - `kernel/integrator/megakernel.h` - 超级内核循环
  - `kernel/integrator/shade_background.h` - 背景着色
  - `kernel/integrator/shade_volume.h` - 体积着色
  - `kernel/device/optix/kernel.cu` - OptiX 内核
  - `kernel/device/gpu/kernel.h` - GPU 内核

## 实现细节 / 关键算法

1. **AO 弹射近似**: 在路径深度超过 `ao_bounces` 时，将射线最大距离缩短到 `ao_bounces_distance`（或对象自定义的 `ao_distance`），用短射线近似间接光照。
2. **MNEE 防火花机制**: 当启用焦散效果时，追踪路径是否经过焦散接收体和投射体。若射线从焦散投射体出发且路径中有接收体祖先，则标记 `PATH_MNEE_CULL_LIGHT_CONNECTION` 以抑制直接光连接（避免火花噪声）。
3. **光源 MIS**: 当 `use_light_mis` 启用时，在场景交点计算后额外执行 `lights_intersect` 查找射线路径上的光源交点，用于多重重要性采样。
4. **自交排除**: 使用上一次交点的 `object` 和 `prim` 设置 `ray.self` 以避免自交。

## 关联文件

- `kernel/bvh/bvh.h` - BVH 遍历引擎
- `kernel/integrator/shade_surface.h` - 表面着色内核
- `kernel/integrator/shade_volume.h` - 体积着色内核
- `kernel/integrator/shade_background.h` - 背景着色内核
- `kernel/integrator/shade_light.h` - 光源着色内核
