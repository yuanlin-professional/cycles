# path_trace.h / path_trace.cpp - 路径追踪核心调度与渲染管线

## 概述

`PathTrace` 是 Cycles 路径追踪渲染器的核心类，负责管理完整的渲染管线（pipeline），包括内核图调度、多设备工作分发、自适应采样、降噪、显示更新以及图块写入等环节。该类对上层 `Session` 提供统一的渲染入口，对下层通过 `PathTraceWork` 将工作分发到各个计算设备（CPU/GPU）上执行。

它协调了从采样到最终像素输出的整个流程，支持多设备并行渲染、分辨率分割（resolution divider）快速预览、路径引导（path guiding）训练与更新、以及渲染取消等功能。

## 类与结构体

### PathTrace

- **功能**: 路径追踪渲染的顶层控制器，管理渲染管线的所有通用步骤
- **关键成员**:
  - `device_` — 用于路径追踪的主设备指针，多设备时为 `MultiDevice`
  - `denoise_device_` — 用于降噪的设备指针
  - `cpu_device_` — CPU 设备，用于创建临时渲染缓冲区
  - `film_` — 胶片/成像平面配置
  - `device_scene_` — 设备侧场景数据
  - `render_scheduler_` — 渲染调度器引用，负责时间和采样决策
  - `tile_manager_` — 图块管理器引用
  - `display_` — `PathTraceDisplay` 智能指针，封装交互式显示驱动
  - `output_driver_` — 输出驱动，用于写入渲染缓冲区
  - `path_trace_works_` — 每个计算设备对应的 `PathTraceWork` 实例列表
  - `work_balance_infos_` — 多设备负载均衡信息
  - `full_params_` / `big_tile_params_` — 全帧和当前大图块的缓冲区参数
  - `denoiser_` — 降噪器实例
  - `render_state_` — 渲染状态（分辨率分割器、参数重置标志、降噪结果标志等）
  - `render_cancel_` — 取消渲染的同步状态（互斥锁 + 条件变量）
  - `progress_` — 进度对象指针
  - `guiding_field_` / `guiding_sample_data_storage_` — 路径引导相关结构（条件编译 `WITH_PATH_GUIDING`）
- **关键方法**:
  - `render(const RenderWork&)` — 阻塞式渲染入口，执行给定数量的采样直至完成或取消
  - `render_pipeline(RenderWork)` — 实际渲染管线实现，按顺序调用各步骤
  - `path_trace(RenderWork&)` — 并行调度所有 `PathTraceWork` 执行采样
  - `adaptive_sample(RenderWork&)` — 执行自适应采样收敛检测与过滤
  - `denoise(const RenderWork&)` — 对大图块执行降噪
  - `update_display(const RenderWork&)` — 将渲染结果更新到显示纹理
  - `rebalance(const RenderWork&)` — 多设备负载再平衡
  - `reset(...)` — 重置渲染状态（如视口导航或尺寸变化时）
  - `cancel()` — 尽快取消渲染，阻塞直到当前 `render_samples()` 返回
  - `copy_to_render_buffers()` / `copy_from_render_buffers()` — 通过 CPU 中转在路径追踪器和外部渲染缓冲区间复制数据
  - `set_denoiser_params()` — 配置降噪器参数，支持按需创建/切换降噪器类型
  - `set_guiding_params()` — 配置路径引导参数
  - `guiding_update_structures()` / `guiding_prepare_structures()` — 更新和准备路径引导结构

## 核心函数

- **`render_pipeline()`**: 渲染管线的核心，按以下顺序执行各步骤（每步之间检查取消状态）:
  1. `render_init_kernel_execution()` — 初始化所有设备队列
  2. `init_render_buffers()` — 初始化渲染缓冲区（清零、读取烘焙数据等）
  3. `rebalance()` — 多设备负载再平衡
  4. `guiding_prepare_structures()` — 准备路径引导结构（如启用）
  5. `path_trace()` — 并行路径追踪采样
  6. `guiding_update_structures()` — 更新路径引导场（如启用训练）
  7. `adaptive_sample()` — 自适应采样过滤与收敛检查
  8. `cryptomatte_postprocess()` — Cryptomatte 后处理
  9. `denoise()` — 降噪
  10. `denoise_volume_guiding_buffers()` — 体积引导缓冲区降噪
  11. `write_tile_buffer()` — 写入图块
  12. `update_display()` — 更新显示
  13. `finalize_full_buffer_on_disk()` — 将完整帧写入磁盘

- **`foreach_sliced_buffer_params()`**: 模板辅助函数，根据负载均衡权重将大图块按行切片分配给各 `PathTraceWork`，支持 overscan 边界扩展

- **`path_trace()`**: 使用 `parallel_for` 并行调度所有 `PathTraceWork::render_samples()`，记录耗时和占用率统计

## 依赖关系

- **内部头文件**:
  - `integrator/denoiser.h`, `integrator/guiding.h`, `integrator/pass_accessor.h`
  - `integrator/path_trace_work.h`, `integrator/work_balancer.h`
  - `session/buffers.h`
  - `util/guiding.h`, `util/thread.h`, `util/unique_ptr.h`, `util/vector.h`
- **实现文件额外依赖**:
  - `device/cpu/device.h`, `device/device.h`
  - `integrator/path_trace_display.h`, `integrator/path_trace_tile.h`, `integrator/render_scheduler.h`
  - `scene/pass.h`, `scene/scene.h`
  - `session/display_driver.h`, `session/tile.h`
  - `util/log.h`, `util/progress.h`, `util/tbb.h`, `util/time.h`
- **被引用**: `session/session.cpp`, `integrator/path_trace_tile.cpp`

## 实现细节 / 关键算法

1. **多设备切片渲染**: 大图块沿 Y 轴按权重切分为若干水平条带，每个 `PathTraceWork` 负责一条。切片支持 overscan 以保证降噪边界质量。

2. **分辨率分割器**: 交互式视口导航时使用较大的 resolution_divider（如 8x）实现快速预览，随后逐步降低到 1x 以获得全分辨率结果。缓冲区参数按分割器缩放，复用同一渲染缓冲区。

3. **取消机制**: `render_cancel_` 结构通过互斥锁和条件变量实现线程安全的取消请求与等待。`cancel()` 设置标志后阻塞等待当前渲染完成，各 `PathTraceWork` 通过 `is_cancel_requested()` 轮询标志以尽快退出。

4. **路径引导**: 条件编译 `WITH_PATH_GUIDING` 下，使用 OpenPGL 库维护全局辐射场（`openpgl::cpp::Field`）和采样数据存储（`openpgl::cpp::SampleStorage`），每次渲染迭代后用训练数据更新引导场。

5. **降噪器管理**: 支持 OIDN（CPU/GPU）和 OptiX 降噪器，根据参数变化动态创建/切换降噪器实例，优先选择支持图形互操作的设备以加速显示更新。

## 关联文件

- `integrator/path_trace_work.h` — 设备级工作抽象基类
- `integrator/path_trace_display.h` — 显示更新封装
- `integrator/render_scheduler.h` — 渲染调度决策
- `integrator/work_balancer.h` — 多设备负载均衡
- `session/session.cpp` — 上层会话管理，创建并驱动 PathTrace
