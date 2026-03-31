# shade_shadow.h - 阴影着色（透明阴影处理）

## 概述

本文件实现了阴影着色内核，处理阴影射线路径上透明表面和体积的着色逻辑。当 `intersect_shadow` 内核检测到透明交点后，本内核遍历这些交点，评估每个透明表面的透明度和体积的衰减，累积最终的阴影通量（throughput）。若所有交点处理完成且路径未完全遮挡，将直接光照贡献写入渲染缓冲区。

## 核心函数

### shadow_intersections_has_remaining()

- **签名**: `ccl_device_inline bool shadow_intersections_has_remaining(const uint num_hits)`
- **功能**: 判断是否还有未记录的阴影交点。当命中数达到或超过 `INTEGRATOR_SHADOW_ISECT_SIZE` 时返回 `true`，表示需要多轮处理。

### integrate_transparent_surface_shadow()

- **签名**: `ccl_device_inline Spectrum integrate_transparent_surface_shadow(KernelGlobals kg, IntegratorShadowState state, const int hit)`
- **功能**: 评估单个透明表面交点的阴影透明度。设置着色器数据，执行着色器评估（使用 `KERNEL_FEATURE_NODE_MASK_SURFACE_SHADOW` 掩码以简化节点评估），处理体积栈进入/退出，检测射线传送门（Portal），最终调用 `surface_shader_transparency` 返回透明度。
- **条件编译**: `__TRANSPARENT_SHADOWS__`

### integrate_transparent_volume_shadow()

- **签名**: `ccl_device_inline void integrate_transparent_volume_shadow(KernelGlobals kg, IntegratorShadowState state, const int hit, const int num_recorded_hits, ccl_private Spectrum *ccl_restrict throughput)`
- **功能**: 评估阴影射线在两个相邻交点之间的体积衰减。根据 `volume_ray_marching` 设置选择光线步进（ray marching）或空散射（null scattering）方法计算体积透明度，直接修改 `throughput`。
- **条件编译**: `__VOLUME__`

### integrate_transparent_shadow()

- **签名**: `ccl_device_inline bool integrate_transparent_shadow(KernelGlobals kg, IntegratorShadowState state, const uint num_hits)`
- **功能**: 处理所有透明阴影交点。遍历每个记录的交点，对每对相邻交点之间的区间：
  1. 评估体积阴影（若体积栈非空）
  2. 评估表面透明度
  3. 累积通量衰减
  4. 若通量为零（完全遮挡）则提前返回 `true`
  5. 递增透明弹射计数和 RNG 偏移
  6. 检查体积边界弹射限制
  7. 若还有未记录的交点，调整射线 `tmin` 以便下一轮继续
- **返回值**: `true` 表示路径被完全遮挡；`false` 表示仍有光线透过。

### integrator_shade_shadow()

- **签名**: `ccl_device void integrator_shade_shadow(KernelGlobals kg, IntegratorShadowState state, ccl_global float *ccl_restrict render_buffer)`
- **功能**: 阴影着色内核的主入口。读取交点命中数，调用 `integrate_transparent_shadow` 处理所有透明表面。根据结果：
  - **完全遮挡**: 终止阴影路径
  - **有剩余交点**: 调度 `INTERSECT_SHADOW` 内核继续查找交点
  - **全部处理完成**: 记录路径引导的直接光照，调用 `film_write_direct_light` 写入渲染缓冲区，然后终止阴影路径

## 依赖关系

- **内部头文件**:
  - `kernel/integrator/guiding.h` - 路径引导直接光照记录
  - `kernel/integrator/shade_volume.h` - 体积阴影计算
  - `kernel/integrator/surface_shader.h` - 表面着色器评估和透明度
  - `kernel/integrator/volume_stack.h` - 体积栈进入/退出
  - `kernel/geom/shader_data.h` - 着色数据设置
- **被引用**:
  - `kernel/integrator/megakernel.h` - 超级内核循环
  - `kernel/device/optix/kernel_osl.cu` - OptiX OSL 内核
  - `kernel/device/gpu/kernel.h` - GPU 内核

## 实现细节 / 关键算法

1. **多轮处理机制**: 受 `INTEGRATOR_SHADOW_ISECT_SIZE` 限制，一次最多记录固定数量的交点。若实际命中数更多，处理完当前批次后调整 `shadow_ray.tmin` 并重新调度 `INTERSECT_SHADOW` 内核。
2. **体积-表面交替评估**: 遍历循环对每个区间先评估体积（若有），再评估表面。额外多一次体积评估（`hit < num_recorded_hits + 1`）以处理世界体积。
3. **射线传送门处理**: 若透明表面被标记为 `SD_RAY_PORTAL`（射线传送门），`integrate_transparent_surface_shadow` 返回零光谱，等效于完全遮挡阴影传送门后的光源。
4. **通量累积**: 每个透明表面的透明度乘以当前通量（`throughput *= shadow`），体积衰减直接修改通量指针。当通量为零时提前退出。
5. **着色器简化**: 使用 `KERNEL_FEATURE_NODE_MASK_SURFACE_SHADOW` 节点掩码，仅评估影响阴影透明度的节点子集，跳过不相关的 BSDF 和位移节点。

## 关联文件

- `kernel/integrator/intersect_shadow.h` - 前序的阴影交点收集内核
- `kernel/film/light_passes.h` - `film_write_direct_light` 最终写入
- `kernel/integrator/shade_volume.h` - 体积阴影评估函数
