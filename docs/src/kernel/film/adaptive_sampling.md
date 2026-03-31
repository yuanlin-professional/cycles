# adaptive_sampling.h - 自适应采样收敛判定与邻域滤波

## 概述

本文件实现了 Cycles 渲染器的自适应采样机制。自适应采样通过逐像素评估蒙特卡洛积分的收敛情况，判断某个像素是否已经达到足够的采样质量，从而跳过后续采样以节省计算资源。此外，文件还包含一个简单的盒式滤波器（box filter），用于在 X/Y 两个方向上扩展未收敛像素的邻域，确保相邻像素也能获得更多采样。

该算法基于论文 "A hierarchical automatic stopping condition for Monte Carlo global illumination" 中的每像素误差估计方法。

## 类与结构体

本文件未定义独立的类或结构体。主要依赖 `kernel_data.film` 中的胶片参数结构体字段：

- `pass_adaptive_aux_buffer` — 自适应采样辅助缓冲区在渲染通道中的偏移量
- `pass_sample_count` — 采样计数通道的偏移量
- `pass_combined` — 合成通道的偏移量
- `pass_stride` — 每像素在渲染缓冲区中的步幅
- `exposure` — 曝光值

## 枚举与常量

- `PASS_UNUSED` — 表示某个渲染通道未被启用的哨兵值

## 核心函数

### film_need_sample_pixel()
- **签名**: `ccl_device_forceinline bool film_need_sample_pixel(KernelGlobals kg, ConstIntegratorState state, ccl_global float *render_buffer)`
- **功能**: 检查某个像素是否仍需要采样。通过读取自适应采样辅助缓冲区的 w 分量（第4个浮点数）来判断：若该值为 0.0f 则表示尚未收敛、仍需采样；若非零则表示已收敛、可跳过。如果自适应采样辅助缓冲区未启用（`PASS_UNUSED`），则始终返回 `true`。

### film_adaptive_sampling_convergence_check()
- **签名**: `ccl_device bool film_adaptive_sampling_convergence_check(KernelGlobals kg, ccl_global float *render_buffer, const int x, const int y, const float threshold, const int reset, const int offset, const int stride)`
- **功能**: 对指定像素执行收敛性检测。核心逻辑为：
  1. 读取辅助缓冲区（A）中的累积值和合成通道（I）中的颜色值
  2. 计算每像素误差 `error_difference = |I - A| * intensity_scale`
  3. 根据亮度进行归一化：低亮度区域使用 `sqrt(intensity)`，高亮度区域使用 `intensity`
  4. 将归一化后的误差与阈值 `threshold` 比较，判定是否收敛
  5. 将收敛结果写入辅助缓冲区的 w 分量

### film_adaptive_sampling_filter_x()
- **签名**: `ccl_device void film_adaptive_sampling_filter_x(KernelGlobals kg, ccl_global float *render_buffer, const int y, const int start_x, const int width, const int offset, const int stride)`
- **功能**: 在 X 方向上执行盒式滤波。当发现一个未收敛的像素时，将其左右相邻的已收敛像素也标记为未收敛，以确保采样边界平滑过渡，避免出现视觉伪影。

### film_adaptive_sampling_filter_y()
- **签名**: `ccl_device void film_adaptive_sampling_filter_y(KernelGlobals kg, ccl_global float *render_buffer, const int x, const int start_y, const int height, const int offset, const int stride)`
- **功能**: 在 Y 方向上执行盒式滤波，逻辑与 X 方向版本对称。两个方向的滤波组合形成两趟盒式滤波，扩展未收敛像素的影响范围。

## 依赖关系

- **内部头文件**:
  - `kernel/film/write.h` — 提供 `film_pass_pixel_render_buffer()` 等渲染缓冲区读写工具函数

- **被引用**:
  - `src/kernel/integrator/init_from_bake.h` — 烘焙初始化时检查像素是否需要采样
  - `src/kernel/integrator/init_from_camera.h` — 相机路径初始化时检查像素是否需要采样
  - `src/kernel/device/gpu/kernel.h` — GPU 内核入口调用收敛检测与滤波函数
  - `src/kernel/device/cpu/kernel_arch_impl.h` — CPU 内核入口调用收敛检测与滤波函数
  - `src/integrator/adaptive_sampling.cpp` — 积分器层面的自适应采样调度
  - `src/integrator/render_scheduler.h` — 渲染调度器中使用自适应采样逻辑
  - `src/scene/integrator.h` — 场景积分器配置
  - `src/test/integrator_adaptive_sampling_test.cpp` — 单元测试

## 实现细节 / 关键算法

### 收敛判定算法

收敛判定基于分层自动停止条件的简化版本：

1. **误差计算**: 将辅助缓冲区中仅使用一半采样（Class A 采样）累积的结果与合成通道的完整结果做差值，得到颜色误差 `error_difference`。
2. **亮度归一化**: 对于亮度 < 1.0 的区域使用 `sqrt(intensity)` 进行归一化，对于亮度 >= 1.0 的区域直接使用 `intensity`。这种设计使得低亮度区域获得更多采样（因为低亮度区域在显示时更能体现细节差异），高亮度/过曝区域获得较少采样。
3. **收敛条件**: 当 `error / (0.0001 + error_normalize) < threshold` 时，判定该像素已收敛。

### 邻域扩展滤波

滤波采用两趟方式（先 X 后 Y），扫描每一行/列的像素：
- 当遇到未收敛像素时，将前一个已收敛像素也标记为未收敛
- 当从未收敛像素切换到已收敛像素时，将该已收敛像素也标记为未收敛

这实质上在未收敛区域的边缘各扩展了一个像素。

## 关联文件

- `src/kernel/film/write.h` — 渲染通道写入基础设施
- `src/kernel/film/light_passes.h` — `film_write_adaptive_buffer()` 函数负责写入自适应采样辅助缓冲区
- `src/integrator/adaptive_sampling.cpp` — CPU 侧自适应采样管理
- `src/kernel/sample/pattern.h` — 提供 `sample_is_class_A()` 函数，用于判断采样分类
