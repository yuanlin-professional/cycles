# intersect_volume_stack.h - 体积栈初始化与更新

## 概述

本文件实现了体积栈的初始化和更新功能。体积栈追踪射线当前所处的所有重叠体积对象，这对于正确计算体积散射、吸收和自发光至关重要。该文件提供两个主要场景的体积栈操作：相机射线的初始体积栈构建（确定相机位于哪些体积内部）以及次表面散射出射点的体积栈更新。

## 核心函数

### integrator_volume_stack_update_for_subsurface()

- **签名**: `ccl_device void integrator_volume_stack_update_for_subsurface(KernelGlobals kg, IntegratorState state, const float3 from_P, const float3 to_P)`
- **功能**: 更新次表面散射入射点到出射点之间的体积栈。从 `from_P` 到 `to_P` 发射一条射线，遍历途中所有体积边界，调用 `volume_stack_enter_exit` 更新体积栈的进入/退出状态。忽略 SSS 自身的对象交点（SSS 本身已处理进出）。
- **实现策略**:
  - `__VOLUME_RECORD_ALL__` 模式：一次性记录所有交点，排序后按距离顺序处理。
  - 逐步遍历模式：迭代推进射线 `tmin`，逐个处理体积边界交点。

### integrator_volume_stack_init()

- **签名**: `ccl_device void integrator_volume_stack_init(KernelGlobals kg, IntegratorState state)`
- **功能**: 初始化相机射线的体积栈。从相机位置沿 Z 轴正方向发射无限长射线，通过分析与体积边界的交点来确定相机位于哪些体积内部。核心逻辑：
  1. 若存在背景体积着色器，将其作为体积栈的第一项（Shadow Catcher 通道除外）
  2. 遍历射线与体积边界的所有交点
  3. 若射线从某体积背面出射但从未从正面进入，说明相机在该体积内部，将其加入栈
  4. 若射线从正面进入某体积，记录该体积并在后续出射时排除
  5. 写入终止标记

### integrator_intersect_volume_stack()

- **签名**: `ccl_device void integrator_intersect_volume_stack(KernelGlobals kg, IntegratorState state)`
- **功能**: 体积栈交点内核的主入口。调用 `integrator_volume_stack_init` 完成初始化后，根据路径类型调度后续内核：
  - **Shadow Catcher 通道**: 调用 `integrator_intersect_next_kernel_after_shadow_catcher_volume` 继续着色
  - **相机射线**: 调度 `INTERSECT_CLOSEST` 内核执行正常的最近交点计算

## 依赖关系

- **内部头文件**:
  - `kernel/bvh/bvh.h` - 层次包围体(BVH)遍历（`scene_intersect_volume`）
  - `kernel/geom/shader_data.h` - 着色数据设置
  - `kernel/integrator/intersect_closest.h` - 最近交点工具函数
  - `kernel/integrator/volume_stack.h` - 体积栈进入/退出操作
- **被引用**:
  - `kernel/integrator/subsurface.h` - 次表面散射中调用体积栈更新
  - `kernel/integrator/megakernel.h` - 超级内核循环
  - `kernel/device/optix/kernel.cu` - OptiX 内核
  - `kernel/device/gpu/kernel.h` - GPU 内核

## 实现细节 / 关键算法

1. **射线方向选择**: 初始化时使用 Z 轴正方向 `(0, 0, 1)` 作为探测射线方向。这是一种启发式选择，旨在最小化交点数量（假设大多数体积沿水平方向更宽）。
2. **背面检测法**: 通过 `SD_BACKFACING` 标志判断射线从体积的哪一侧穿过。若射线从背面出射且该体积不在"已进入"列表中，说明相机在体积内部。
3. **去重机制**: 在添加体积到栈之前，检查是否已经存在于当前栈中和"已封闭"列表中，避免重复记录。
4. **栈大小限制**: 体积栈大小受 `MAX_VOLUME_STACK_SIZE` 限制，遍历步数限制为 `2 * volume_stack_size`。
5. **CUDA 兼容性**: 由于 CUDA 不支持变长数组，使用 `MAX_VOLUME_STACK_SIZE` 作为固定大小数组。

## 关联文件

- `kernel/integrator/volume_stack.h` - 体积栈进入/退出操作核心逻辑
- `kernel/integrator/intersect_closest.h` - 提供 Shadow Catcher 相关的调度函数
- `kernel/integrator/subsurface.h` - 次表面散射中调用 `integrator_volume_stack_update_for_subsurface`
- `kernel/integrator/shade_volume.h` - 体积着色内核
