# intersect_dedicated_light.h - 阴影链接专用光源交点计算

## 概述

本文件实现了阴影链接（Shadow Linking）功能中专用光源的交点计算逻辑。阴影链接允许艺术家控制哪些对象为哪些光源投射阴影，从而实现非物理但艺术上期望的光影效果。该内核负责沿射线方向查找被阴影链接排除的发光体，并将交点信息传递给专用光源着色内核。

## 核心函数

### shadow_linking_pick_mesh_intersection()

- **签名**: `ccl_device int shadow_linking_pick_mesh_intersection(KernelGlobals kg, IntegratorState state, ccl_private Ray *ccl_restrict ray, const int object_receiver, ccl_private Intersection *ccl_restrict linked_isect, ccl_private uint *lcg_state, int num_hits)`
- **功能**: 沿射线搜索网格对象中的发光表面。使用蓄水池采样（Reservoir Sampling）在多个命中中随机选取一个。检查对象的 `shadow_set_membership` 是否不等于 `LIGHT_LINK_MASK_ALL`（即被排除于默认阴影集合之外）。遇到非透明默认遮挡体时停止搜索并设置 `ray->tmax`。
- **返回值**: 累计命中的发光表面数量。

### shadow_linking_pick_light_intersection()

- **签名**: `ccl_device bool shadow_linking_pick_light_intersection(KernelGlobals kg, IntegratorState state, ccl_private Ray *ccl_restrict ray, ccl_private Intersection *ccl_restrict linked_isect)`
- **功能**: 综合搜索网格发光体和分析光源，选取一个用于阴影链接的光源交点。先调用 `shadow_linking_pick_mesh_intersection` 搜索网格，再调用 `lights_intersect_shadow_linked` 搜索分析光源。将选中光源的权重（命中数）写入积分器状态。
- **返回值**: `true` 表示找到了需要阴影链接的光源；`false` 表示无光源命中。

### shadow_linking_intersect()

- **签名**: `ccl_device bool shadow_linking_intersect(KernelGlobals kg, IntegratorState state)`
- **功能**: 阴影链接交点计算的顶层入口。读取射线状态，调用 `shadow_linking_pick_light_intersection` 查找光源，保存主路径的自交检查图元，将光源交点写入积分器状态，并调度 `SHADE_DEDICATED_LIGHT` 内核。
- **返回值**: `true` 表示已调度专用光源着色；`false` 表示无需额外阴影射线。

### integrator_intersect_dedicated_light()

- **签名**: `ccl_device void integrator_intersect_dedicated_light(KernelGlobals kg, IntegratorState state)`
- **功能**: 专用光源交点内核的主入口。调用 `shadow_linking_intersect` 尝试进行阴影链接。若不需要阴影链接，直接调用 `integrator_shade_surface_next_kernel` 继续主路径。整个功能在 `__SHADOW_LINKING__` 编译宏保护下。

## 依赖关系

- **内部头文件**:
  - `kernel/bvh/bvh.h` - 层次包围体(BVH)遍历
  - `kernel/integrator/path_state.h` - 路径状态
  - `kernel/integrator/shade_surface.h` - 表面着色（提供 `integrator_shade_surface_next_kernel`）
  - `kernel/integrator/shadow_linking.h` - 阴影链接工具函数
  - `kernel/light/light.h` - 光源交点计算
  - `kernel/sample/lcg.h` - 线性同余生成器（用于蓄水池采样）
- **被引用**:
  - `kernel/integrator/megakernel.h` - 超级内核循环
  - `kernel/device/optix/kernel.cu` - OptiX 内核
  - `kernel/device/gpu/kernel.h` - GPU 内核

## 实现细节 / 关键算法

1. **蓄水池采样**: 使用 LCG 随机数生成器在遍历过程中对命中的发光体进行等概率随机选择，无需预知命中总数。概率为 `1/num_hits`。
2. **遮挡体检测**: 当射线遇到一个不透明的默认遮挡体（`blocker_shadow_set == 0` 且非纯体积/透明阴影着色器）时，将 `ray->tmax` 设置为该交点距离，之后的光源不再参与阴影链接计算。
3. **透明弹射限制**: 透明表面和体积边界各自计数，超过限制时视为完全遮挡。
4. **最大交点限制**: `SHADOW_LINK_MAX_INTERSECTION_COUNT = 1024`，防止在复杂场景中无限循环。
5. **状态保存与恢复**: 在写入光源交点前，通过 `shadow_linking_store_last_primitives` 保存主路径的自交检查图元，在着色完成后由 `shadow_linking_restore_last_primitives` 恢复。

## 关联文件

- `kernel/integrator/shade_dedicated_light.h` - 专用光源着色内核
- `kernel/integrator/shadow_linking.h` - 阴影链接辅助函数
- `kernel/integrator/shade_surface.h` - 表面着色和后续内核调度
