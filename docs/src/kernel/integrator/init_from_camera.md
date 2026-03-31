# init_from_camera.h - 相机射线路径初始化

## 概述

本文件实现了从相机初始化路径追踪状态的功能，是渲染管线中最常见的路径起点。它负责生成相机射线（包括滤波采样、运动模糊和景深），初始化积分器路径状态，并调度后续的交点计算内核。该文件是波前路径追踪架构中的入口内核之一。

## 核心函数

### integrate_camera_sample()

- **签名**: `ccl_device_inline Spectrum integrate_camera_sample(KernelGlobals kg, const int sample, const int x, const int y, const uint rng_pixel, ccl_private Ray *ray)`
- **功能**: 生成相机采样射线。执行以下步骤：
  1. 生成像素滤波采样偏移（第一个样本使用像素中心 `(0.5, 0.5)`）
  2. 生成运动模糊时间采样和景深镜头采样
  3. 调用 `camera_sample` 生成最终射线
- **返回值**: 相机射线的光谱权重（透过率），零值表示射线被相机拒绝。
- **采样策略**: 使用 `x` 分量作为时间、`y/z` 分量作为镜头坐标，这种安排在 Sobol 采样下对同时具有运动模糊和景深的对象提供更好的收敛效果。

### integrator_init_from_camera()

- **签名**: `ccl_device bool integrator_init_from_camera(KernelGlobals kg, IntegratorState state, const ccl_global KernelWorkTile *ccl_restrict tile, ccl_global float *render_buffer, const int x, const int y, const int scheduled_sample)`
- **功能**: 从相机初始化路径追踪状态。具体步骤：
  1. 初始化基本路径状态（像素位置、渲染缓冲区索引）
  2. 检查像素是否已收敛（自适应采样）
  3. 计算有效采样索引并记录样本
  4. 初始化随机数种子
  5. 调用 `integrate_camera_sample` 生成相机射线
  6. 将射线写入积分器状态
  7. 初始化路径积分状态（弹射计数、通道标志等）
  8. 调度后续内核：若相机在体积内部则先执行 `INTERSECT_VOLUME_STACK`，否则直接执行 `INTERSECT_CLOSEST`
- **返回值**: `false` 表示像素已收敛；`true` 表示路径已成功初始化。

## 依赖关系

- **内部头文件**:
  - `kernel/camera/camera.h` - 相机射线生成（`camera_sample`）
  - `kernel/film/adaptive_sampling.h` - 自适应采样收敛检查
  - `kernel/film/light_passes.h` - 样本计数和缓冲区写入
  - `kernel/integrator/path_state.h` - 路径状态初始化
  - `kernel/integrator/state_util.h` - 积分器状态工具
  - `kernel/sample/pattern.h` - 采样模式和 RNG
- **被引用**:
  - `kernel/device/gpu/kernel.h` - GPU 内核入口
  - `kernel/device/cpu/kernel_arch_impl.h` - CPU 架构实现
  - `kernel/device/optix/kernel_osl_camera.cu` - OptiX OSL 相机内核

## 实现细节 / 关键算法

1. **自适应采样集成**: 通过 `film_need_sample_pixel` 检查像素收敛状态，使 CPU 实现能够在像素收敛后跳过后续采样。
2. **样本偏移**: `film_write_sample` 支持在像素收敛后从其他位置添加样本的逻辑，`scheduled_sample` 可能与实际像素样本数不同。
3. **体积栈初始化**: 当 `kernel_data.cam.is_inside_volume` 为真时，相机可能位于体积内部，需要先执行 `INTERSECT_VOLUME_STACK` 内核来建立正确的体积栈，然后再进行最近交点计算。
4. **零权重射线处理**: 若 `integrate_camera_sample` 返回零光谱（射线被拒绝），函数仍返回 `true` 以确保样本计数正确。

## 关联文件

- `kernel/camera/camera.h` - 相机模型和射线生成
- `kernel/integrator/intersect_closest.h` - 后续的最近交点计算内核
- `kernel/integrator/intersect_volume_stack.h` - 体积栈初始化内核
- `kernel/integrator/path_state.h` - 路径状态管理
