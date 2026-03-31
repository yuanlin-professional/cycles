# render_scheduler.h / render_scheduler.cpp - 渲染调度器

## 概述

`RenderScheduler` 是 Cycles 渲染过程的智能调度器，负责决定每次渲染迭代的具体工作内容，包括采样数量、分辨率分割、自适应采样阈值、降噪时机、显示更新频率以及多设备负载再平衡等。它通过收集和分析渲染时间统计信息，动态调整调度策略以平衡渲染效率和交互响应性。

该类同时定义了 `RenderWork` 数据结构，用于描述一次渲染迭代需要执行的所有任务。`RenderScheduler` 不直接了解设备细节，而是通过 `PathTrace` 上报的时间信息进行决策。

## 类与结构体

### RenderWork

- **功能**: 描述单次渲染迭代需要执行的所有工作项
- **关键成员**:
  - `resolution_divider` — 分辨率分割器（1=全分辨率，更大值用于快速预览）
  - `init_render_buffers` — 是否需要初始化（清零）渲染缓冲区
  - `path_trace` — 路径追踪采样参数（`start_sample`, `num_samples`, `sample_offset`）
  - `adaptive_sampling` — 自适应采样参数（`filter`, `threshold`, `reset`）
  - `cryptomatte` — Cryptomatte 后处理标志
  - `tile` — 图块相关工作（`write` 写入、`denoise` 降噪）
  - `full` — 全帧相关工作（`write` 写入完整渲染结果）
  - `display` — 显示更新（`update` 更新标志、`use_denoised_result` 是否使用降噪结果）
  - `rebalance` — 是否执行多设备负载再平衡
  - `operator bool()` — 转换为 bool，判断是否有任何待执行工作

### RenderScheduler

- **功能**: 渲染工作调度决策引擎
- **关键成员**:
  - `state_` — 调度状态（已渲染采样数、最后更新时间、再平衡状态、自适应阈值等）
  - `first_render_time_` — 首次全分辨率渲染的时间信息（用于估算分辨率分割器）
  - `path_trace_time_` / `denoise_time_` / `display_update_time_` 等 — 各环节的时间统计（`TimeWithAverage`）
  - `tile_manager_` — 图块管理器引用
  - `buffer_params_` — 当前缓冲区参数
  - `denoiser_params_` — 降噪器参数
  - `adaptive_sampling_` — 自适应采样配置
  - `headless_` / `background_` — 无界面/后台渲染标志
  - `pixel_size_` — 像素大小（高 DPI 显示）
  - `sample_offset_` / `num_samples_` — 采样范围
  - `time_limit_` — 渲染时间限制（分钟）
  - `use_progressive_noise_floor_` — 是否使用渐进式噪声底限
  - `limit_samples_per_update_` — 每次更新的采样数限制（路径引导用）
- **关键方法**:
  - `get_render_work()` — 获取下一个渲染工作（核心调度入口）
  - `reset(buffer_params)` — 重置调度器
  - `reset_for_next_tile()` — 切换到下一个图块时重置
  - `set_sample_params(...)` — 设置采样范围参数，支持子集渲染
  - `report_path_trace_time()` / `report_denoise_time()` 等 — 上报各环节耗时
  - `report_path_trace_occupancy()` — 上报设备占用率
  - `render_work_reschedule_on_converge()` — 收敛后重新调度（降低阈值继续）
  - `render_work_reschedule_on_idle()` — 设备空闲时重新调度
  - `render_work_reschedule_on_cancel()` — 取消时调整工作（跳过不必要步骤）
  - `full_report()` — 生成完整的渲染报告

### RenderScheduler::TimeWithAverage

- **功能**: 时间统计辅助类，同时追踪总挂壁时间和加权平均时间
- **关键成员**: `total_wall_time_`, `average_time_accumulator_`, `num_average_times_`, `last_sample_time_`
- **关键方法**: `add_wall()`, `add_average()`, `get_wall()`, `get_average()`, `get_last_sample_time()`, `reset_average()`

## 核心函数

- **`get_render_work()`**: 调度决策的核心函数，流程如下：
  1. 若所有工作已完成（`done()`），检查后处理和全帧写入
  2. 计算分辨率分割器（`update_start_resolution_divider()`）
  3. 确定路径追踪采样数（`get_num_samples_to_path_trace()`）
  4. 决定是否需要自适应采样过滤
  5. 决定是否需要降噪（`work_need_denoise()`）
  6. 决定是否需要更新显示（`work_need_update_display()`）
  7. 决定是否需要负载再平衡（`work_need_rebalance()`）
  8. 调用 `update_state_for_render_work()` 更新状态

- **`guess_display_update_interval_in_seconds()`**: 启发式计算显示更新间隔——低采样数时更频繁（约 50ms），高采样数时更稀疏（数秒），以平衡视觉反馈和设备占用率。

- **`calculate_num_samples_per_update()`**: 根据平均采样时间和目标更新间隔，计算每次更新的采样数。确保至少 1 个采样，且不超过剩余采样数。

- **`update_start_resolution_divider()`**: 根据首次渲染时间估算合适的分辨率分割器，使视口导航时的总渲染+降噪+显示时间保持在理想范围内。

## 依赖关系

- **内部头文件**:
  - `integrator/adaptive_sampling.h`, `integrator/denoiser.h`
  - `session/buffers.h`
  - `util/string.h`
- **实现文件额外依赖**:
  - `scene/integrator.h`
  - `session/session.h`, `session/tile.h`
  - `util/log.h`, `util/time.h`
- **被引用**: `integrator/path_trace.cpp`, `session/session.h`, `test/integrator_render_scheduler_test.cpp`

## 实现细节 / 关键算法

1. **渐进式噪声底限（Progressive Noise Floor）**: 交互式渲染时，自适应采样阈值从高值逐步降低，使图像以均匀噪声等级渐进细化。收敛后自动降低阈值继续采样，直到达到最终阈值。

2. **分辨率分割器自动计算**: 假设渲染时间与像素数线性相关（与分割器的平方成反比）。根据首次全分辨率渲染时间和期望的导航更新间隔，反算出合适的分割器值。

3. **子集渲染**: 支持跨多台计算机分布渲染单帧。通过 `sample_offset` 和 `num_samples` 配合子集偏移和长度，各节点渲染不重叠的采样范围。

4. **时间限制**: 设置 `time_limit_` 后，调度器会在超时时标记 `path_trace_finished`，但仍会安排必要的降噪和写入工作以输出最终结果。

5. **负载再平衡时机**: 默认在渲染初期（前 2 次再平衡请求）进行，之后每 2 秒检查一次。如果最近的再平衡未产生实际变化，则延长间隔。

6. **降噪调度**: 降噪频率根据渲染采样数动态调整——初期每隔较少采样即降噪以提供快速反馈，后期延长降噪间隔以减少开销。考虑 GPU 降噪和 CPU 降噪的性能差异。

## 全局辅助函数

- `calculate_resolution_divider_for_resolution(width, height, resolution)` — 根据目标分辨率计算分割器
- `calculate_resolution_for_divider(width, height, resolution_divider)` — 根据分割器计算实际分辨率

## 关联文件

- `integrator/path_trace.h` — 上层路径追踪器，调用调度器获取工作
- `session/session.h` — 会话持有调度器
- `integrator/adaptive_sampling.h` — 自适应采样参数
- `session/tile.h` — 图块管理
