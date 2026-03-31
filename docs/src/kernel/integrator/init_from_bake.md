# init_from_bake.h - 烘焙模式路径初始化

## 概述

本文件实现了从烘焙（Bake）操作初始化路径追踪状态的功能。烘焙是将着色结果写入纹理的过程，不同于从相机发射射线，烘焙从网格表面上的纹素中心点发射射线。该文件处理重心坐标抖动抗锯齿、UV 坐标转换、射线设置以及后续内核调度，支持环境烘焙和表面烘焙两种模式。

## 核心函数

### bake_jitter_barycentric()

- **签名**: `ccl_device_inline void bake_jitter_barycentric(ccl_private float &u, ccl_private float &v, float2 rand_filter, const float dudx, const float dudy, const float dvdx, const float dvdy)`
- **功能**: 在烘焙过程中对输入的重心坐标进行抖动以实现抗锯齿。通过微分（dudx、dudy、dvdx、dvdy）将滤波采样偏移映射到重心空间。若抖动后的坐标落在三角形外部，最多重试 10 次（使用哈希生成新的抖动值）。若始终无法落入三角形内部，则放弃抖动使用原始中心值。

### bake_offset_towards_center()

- **签名**: `ccl_device float2 bake_offset_towards_center(KernelGlobals kg, const int prim, const float u, const float v)`
- **功能**: 当采样点恰好位于三角形顶点时，将其轻微偏移向三角形中心以避免光线追踪精度问题（自交）。使用经验确定的偏移量（位置偏移 `1e-4f`，UV 偏移 `1e-5f`），通过重心坐标插值计算偏移后的新 UV。

### integrator_init_from_bake()

- **签名**: `ccl_device bool integrator_init_from_bake(KernelGlobals kg, IntegratorState state, const ccl_global KernelWorkTile *ccl_restrict tile, ccl_global float *render_buffer, const int x, const int y, const int scheduled_sample)`
- **功能**: 从烘焙工作块初始化积分器路径状态。该函数是烘焙管线的入口点，执行以下步骤：
  1. 初始化基本路径状态并检查像素收敛情况
  2. 从渲染缓冲区读取图元索引和重心坐标
  3. 设置随机数生成器（支持自定义种子）
  4. 执行重心坐标抖动抗锯齿
  5. 将 Blender 重心坐标转换为 Cycles/Embree/OptiX 约定
  6. 计算三角形上的位置和法线
  7. 根据烘焙类型设置射线并调度后续内核
- **返回值**: `false` 表示像素已收敛不需要继续采样；`true` 表示已初始化路径。

## 依赖关系

- **内部头文件**:
  - `kernel/camera/camera.h` - 相机方向计算（用于 `camera_direction_from_point`）
  - `kernel/film/adaptive_sampling.h` - 自适应采样
  - `kernel/film/light_passes.h` - 光线通道写入
  - `kernel/integrator/intersect_closest.h` - 交点计算工具函数
  - `kernel/integrator/path_state.h` - 路径状态初始化
  - `kernel/sample/pattern.h` - 采样模式
- **被引用**:
  - `kernel/device/gpu/kernel.h` - GPU 内核入口
  - `kernel/device/cpu/kernel_arch_impl.h` - CPU 架构实现

## 实现细节 / 关键算法

1. **环境烘焙**: 当 `pass_background` 不为 `PASS_UNUSED` 时，从原点沿位置方向发射射线，直接调度 `SHADE_BACKGROUND` 内核。
2. **表面烘焙**: 从三角形表面沿法线（或相机方向）偏移一定距离发射反向射线。支持 `use_camera` 选项，此时射线方向基于相机位置，并对法线进行平滑偏移以避免背面出现纯黑。
3. **快速通道**: 对于位置通道（`pass_position`）和无凹凸贴图的法线通道（`pass_normal`），直接写入几何数据，跳过完整着色。
4. **重心坐标转换**: Blender 使用 `(u, v)` 表示三角形上的位置，而 Cycles/Embree/OptiX 使用不同约定。转换公式为：`u_new = v_old`, `v_new = 1 - u_old - v_old`。
5. **内核调度**: 根据着色器标志选择 `SHADE_SURFACE`、`SHADE_SURFACE_RAYTRACE` 或 `SHADE_SURFACE_MNEE` 内核。
6. **Shadow Catcher**: 若启用，在路径初始化后调用 `integrator_split_shadow_catcher` 分裂路径。

## 关联文件

- `kernel/integrator/intersect_closest.h` - 提供 `integrator_split_shadow_catcher` 等工具函数
- `kernel/integrator/path_state.h` - 路径状态初始化函数
- `kernel/device/gpu/kernel.h` - GPU 内核调度入口
- `kernel/device/cpu/kernel_arch_impl.h` - CPU 内核架构实现
