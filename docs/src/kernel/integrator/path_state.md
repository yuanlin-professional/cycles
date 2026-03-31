# path_state.h - 路径状态管理与随机数采样

## 概述

本文件是路径追踪积分器的核心状态管理模块，负责路径状态的初始化、弹射计数更新、路径终止判断（俄罗斯轮盘赌）、射线可见性标志计算以及随机数生成。它定义了 `RNGState` 结构和一系列用于多维采样的工具函数，确保路径中每个步骤使用唯一且不相关的随机数序列。

## 核心函数

### path_state_init_queues()

- **签名**: `ccl_device_inline void path_state_init_queues(IntegratorState state)`
- **功能**: 初始化内核队列为零，将路径标记为已终止。用于相机射线初始化的早期退出和 Shadow Catcher 分裂状态的初始化。

### path_state_init()

- **签名**: `ccl_device_inline void path_state_init(IntegratorState state, const ccl_global KernelWorkTile *ccl_restrict tile, const int x, const int y)`
- **功能**: 最小化路径状态初始化。设置渲染像素索引并调用 `path_state_init_queues`，为早期输出提供基本的缓冲区访问能力。

### path_state_init_integrator()

- **签名**: `ccl_device_inline void path_state_init_integrator(KernelGlobals kg, IntegratorState state, const int sample, const uint rng_pixel, const Spectrum throughput)`
- **功能**: 初始化完整的路径积分状态。设置所有弹射计数为零，配置初始路径标志（`PATH_RAY_CAMERA | PATH_RAY_MIS_SKIP | PATH_RAY_TRANSPARENT_BACKGROUND`），初始化通量、MIS PDF、继续概率，以及路径引导、MNEE、去噪、光链接等可选特性的状态。同时初始化体积栈的背景体积条目。

### path_state_next()

- **签名**: `ccl_device_inline void path_state_next(KernelGlobals kg, IntegratorState state, const int label, const int shader_flag)`
- **功能**: 在每次表面/体积弹射后更新路径状态。根据弹射标签处理：
  - **透明/射线传送门**: 增加透明弹射计数，不计入常规弹射
  - **体积散射**: 增加体积弹射计数，设置 `PATH_RAY_VOLUME_SCATTER`
  - **表面反射**: 区分漫反射和光泽反射，分别增加弹射计数
  - **表面透射**: 增加透射弹射计数
  - 每次弹射增加 `rng_offset` 以确保采样维度唯一

### path_state_volume_next()

- **签名**: `ccl_device_inline bool path_state_volume_next(IntegratorState state)`
- **功能**: 处理体积边界网格的穿越。增加 `volume_bounds_bounce` 计数，超过 `VOLUME_BOUNDS_MAX` 时返回 `false`（防止自交导致的无限循环）。

### path_state_ray_visibility()

- **签名**: `ccl_device_inline uint path_state_ray_visibility(ConstIntegratorState state)`
- **功能**: 从路径标志计算射线可见性掩码。去除透射路径中的漫反射/光泽可见性位，并处理 Shadow Catcher 可见性偏移。

### path_state_continuation_probability()

- **签名**: `ccl_device_inline float path_state_continuation_probability(KernelGlobals kg, ConstIntegratorState state, const uint32_t path_flag)`
- **功能**: 计算路径继续概率（用于俄罗斯轮盘赌）。在最小弹射次数内返回 1.0（不终止），之后基于路径通量的最大分量的平方根返回终止概率，上限为 1.0。若启用路径引导，还会考虑未引导通量。

### path_state_ao_bounce()

- **签名**: `ccl_device_inline bool path_state_ao_bounce(KernelGlobals kg, ConstIntegratorState state)`
- **功能**: 判断当前路径是否处于 AO 弹射近似阶段。有效弹射数 = 总弹射 - 透射弹射 - (有光泽弹射 ? 1 : 0) + 1，超过 `ao_bounces` 时返回 `true`。

### path_state_rng_load() / shadow_path_state_rng_load()

- **功能**: 从积分器状态加载 `RNGState` 到栈上，避免频繁的全局内存访问。

### path_state_rng_scramble()

- **签名**: `ccl_device_inline void path_state_rng_scramble(ccl_private RNGState *rng_state, const int seed)`
- **功能**: 通过哈希修改 `rng_offset` 获取不相关的采样序列（如次表面随机游走），确保偏移量为 4 的倍数。

### path_state_rng_1D() / path_state_rng_2D() / path_state_rng_3D()

- **功能**: 从 RNG 状态和指定维度生成 1/2/3 维随机数。

### path_branched_rng_1D() / path_branched_rng_2D() / path_branched_rng_3D()

- **功能**: 为分支路径生成随机数，通过 `sample * num_branches + branch` 偏移保证分支间的采样不相关。

### path_state_rng_light_termination()

- **签名**: `ccl_device_inline float path_state_rng_light_termination(KernelGlobals kg, const ccl_private RNGState *state)`
- **功能**: 获取光源终止随机数。仅当 `light_inv_rr_threshold > 0` 时返回非零值，否则返回 0（禁用光源 RR）。

## 依赖关系

- **内部头文件**:
  - `kernel/integrator/state.h` - 积分器状态定义
  - `kernel/sample/pattern.h` - 底层采样模式（`path_rng_1D`、`path_rng_2D` 等）
- **被引用**:
  - `kernel/integrator/init_from_bake.h` - 烘焙初始化
  - `kernel/integrator/init_from_camera.h` - 相机初始化
  - `kernel/integrator/intersect_closest.h` - 交点计算和路径终止
  - `kernel/integrator/intersect_dedicated_light.h` - 专用光源
  - `kernel/integrator/shade_surface.h` - 表面着色
  - `kernel/integrator/shade_volume.h` - 体积着色
  - `kernel/integrator/subsurface.h` / `subsurface_disk.h` / `subsurface_random_walk.h` - 次表面散射
  - `kernel/svm/ao.h` / `kernel/svm/bevel.h` - SVM 节点
  - `integrator/path_trace_work_cpu.cpp` - CPU 路径追踪

## 实现细节 / 关键算法

1. **RNGState 结构**: 包含 `rng_pixel`（像素级种子）、`rng_offset`（维度偏移）和 `sample`（样本索引），三者共同确定唯一的随机数序列位置。
2. **维度偏移管理**: 每次弹射增加 `PRNG_BOUNCE_NUM` 维度偏移，每个弹射内的不同用途（BSDF 采样、光源采样等）使用 `rng_offset + dimension` 访问不同维度。
3. **俄罗斯轮盘赌优化**: 使用 `sqrt(max_component(throughput))` 作为继续概率。平方根近似匹配典型的视图变换曲线，延迟路径终止以减少方差。
4. **路径标志位系统**: 使用位掩码系统追踪路径类型（相机、反射、透射、漫反射、光泽等），支持可见性过滤、渲染通道分类和各种优化判断。

## 关联文件

- `kernel/sample/pattern.h` - 底层采样模式实现（Sobol、蓝噪声等）
- `kernel/integrator/state.h` - 积分器状态宏定义
- `kernel/integrator/intersect_closest.h` - 路径终止判断的主要消费者
