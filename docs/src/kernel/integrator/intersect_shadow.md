# intersect_shadow.h - 阴影射线交点计算

## 概述

本文件实现了阴影射线的交点计算内核，用于判断光源与着色点之间是否存在遮挡物。支持两种模式：不透明阴影（仅检查是否存在任意遮挡）和透明阴影（记录所有透明交点以便后续着色器评估）。该内核是直接光照计算管线中的关键环节。

## 核心函数

### integrate_intersect_shadow_visibility()

- **签名**: `ccl_device_forceinline uint integrate_intersect_shadow_visibility(ConstIntegratorShadowState state)`
- **功能**: 计算阴影射线的可见性标志。基础可见性为 `PATH_RAY_SHADOW`，若路径处于 Shadow Catcher 通道，则使用移位后的高位可见性标志。

### integrate_intersect_shadow_opaque()

- **签名**: `ccl_device bool integrate_intersect_shadow_opaque(KernelGlobals kg, IntegratorShadowState state, const ccl_private Ray *ray, const uint visibility)`
- **功能**: 执行不透明阴影交点测试。仅检查射线路径上是否存在任何不透明遮挡物。若未命中，将 `num_hits` 设为 0 以通知后续着色内核。
- **返回值**: `true` 表示有不透明遮挡（阴影路径终止）；`false` 表示无遮挡。

### integrate_shadow_max_transparent_hits()

- **签名**: `ccl_device_forceinline int integrate_shadow_max_transparent_hits(KernelGlobals kg, ConstIntegratorShadowState state)`
- **功能**: 计算当前路径允许的最大透明命中数，等于 `transparent_max_bounce - transparent_bounce`。

### shadow_intersections_compare()

- **签名**: `ccl_device int shadow_intersections_compare(const void *a, const void *b)`
- **功能**: CPU 端用于 `qsort` 的交点比较函数，按交点距离 `t` 排序。

### sort_shadow_intersections()

- **签名**: `ccl_device_inline void sort_shadow_intersections(IntegratorShadowState state, uint num_hits)`
- **功能**: 对阴影交点按距离排序。GPU 端使用冒泡排序（更友好的内存访问模式），CPU 端使用 `qsort`。

### integrate_intersect_shadow_transparent()

- **签名**: `ccl_device bool integrate_intersect_shadow_transparent(KernelGlobals kg, IntegratorShadowState state, const ccl_private Ray *ray, const uint visibility)`
- **功能**: 执行透明阴影交点收集。调用 `scene_intersect_shadow_all` 记录射线路径上的所有交点（最多 `max_transparent_hits` 个）。处理预烘焙阴影透明度的通量衰减，对记录的交点按距离排序。
- **返回值**: `true` 表示存在不透明遮挡；`false` 表示所有交点均为透明。

### integrator_intersect_shadow()

- **签名**: `ccl_device void integrator_intersect_shadow(KernelGlobals kg, IntegratorShadowState state)`
- **功能**: 阴影射线交点的主入口函数。读取阴影射线和自交排除信息，计算可见性，根据 `transparent_shadows` 设置选择透明或不透明交点模式。若为不透明命中则终止阴影路径，否则调度 `SHADE_SHADOW` 内核处理透明表面。

## 依赖关系

- **内部头文件**:
  - `kernel/globals.h` - 内核全局数据
  - `kernel/types.h` - 内核类型
  - `kernel/bvh/bvh.h` - 层次包围体(BVH)遍历
  - `kernel/integrator/state.h` - 积分器状态
  - `kernel/integrator/state_flow.h` - 状态流控制
  - `kernel/integrator/state_util.h` - 状态工具函数
- **被引用**:
  - `kernel/integrator/megakernel.h` - 超级内核循环
  - `kernel/device/optix/kernel.cu` - OptiX 内核
  - `kernel/device/gpu/kernel.h` - GPU 内核

## 实现细节 / 关键算法

1. **不透明优先**: `integrate_intersect_shadow_opaque` 仅使用 `PATH_RAY_SHADOW_OPAQUE` 可见性掩码，只检测不透明几何体，效率最高。
2. **透明阴影**: `scene_intersect_shadow_all` 一次性收集所有透明交点。受 `INTEGRATOR_SHADOW_ISECT_SIZE` 限制记录数量，超出部分需通过多次内核调度处理。
3. **排序策略差异化**: GPU 使用冒泡排序以获得更连续的内存访问模式，CPU 使用标准库 `qsort` 获得更好的渐进性能。
4. **预烘焙透明度**: `scene_intersect_shadow_all` 可直接输出预烘焙的透明度通量（`throughput`），绕过着色器评估以提升性能。
5. **编译时可见性掩码**: 使用 `constexpr` 在编译时计算不透明可见性掩码，涵盖 Shadow Catcher 和常规路径两种情况。

## 关联文件

- `kernel/integrator/shade_shadow.h` - 后续的阴影着色内核（处理透明表面）
- `kernel/bvh/bvh.h` - BVH 遍历引擎（`scene_intersect_shadow`、`scene_intersect_shadow_all`）
- `kernel/integrator/state_flow.h` - 阴影路径的状态流转控制
