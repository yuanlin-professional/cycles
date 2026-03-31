# session.h / session.cpp - 渲染会话控制与生命周期管理

## 概述
本文件定义了 Cycles 渲染器的核心会话（Session）类，是整个渲染流程的中枢控制器。Session 管理渲染线程的生命周期，协调场景更新、路径追踪执行、降噪处理和显示输出等各个子系统。它封装了渲染设备创建、瓦片调度、渐进式渲染、暂停/恢复、取消等完整的渲染控制逻辑，是宿主应用（如 Blender）与 Cycles 渲染引擎之间的主要交互入口。

## 类与结构体

### SessionParams
- **功能**: 渲染会话的配置参数，决定设备选择、采样设置、分块策略等全局渲染行为。
- **关键成员**:
  - `device` (`DeviceInfo`) — 渲染设备信息（基于 Blender 偏好设置、场景配置和命令行参数选择）
  - `denoise_device` (`DeviceInfo`) — 降噪设备信息（可与渲染设备相同或不同）
  - `headless` (`bool`) — 是否无头模式（无显示）
  - `background` (`bool`) — 是否后台渲染
  - `samples` (`int`) — 目标采样数，默认 1024
  - `use_sample_subset` (`bool`) — 是否使用采样子集
  - `sample_subset_offset` / `sample_subset_length` (`int`) — 采样子集的偏移和长度
  - `pixel_size` (`int`) — 像素大小
  - `threads` (`int`) — 线程数（0 表示自动）
  - `time_limit` (`double`) — 路径追踪时间限制（秒），0 表示无限制
  - `use_profiling` (`bool`) — 是否启用性能分析
  - `use_auto_tile` (`bool`) — 是否自动分块，默认 `true`
  - `tile_size` (`int`) — 瓦片大小，默认 2048
  - `use_resolution_divider` (`bool`) — 是否使用分辨率除数（渐进式渲染）
  - `shadingsystem` (`ShadingSystem`) — 着色系统选择，默认 `SHADINGSYSTEM_SVM`（着色器虚拟机）
  - `temp_dir` (`string`) — 临时目录（存放进行中的 EXR 瓦片文件）
- **关键方法**:
  - `modified(params)` — 判断参数变化是否需要重建会话（如设备、线程数、着色系统等核心参数变化）

### Session
- **功能**: 渲染会话主类，管理整个渲染流程的生命周期。
- **公开成员**:
  - `device` (`unique_ptr<Device>`) — 渲染设备
  - `denoise_device_` (`unique_ptr<Device>`) — 降噪设备（可能与渲染设备相同）
  - `scene` (`unique_ptr<Scene>`) — 场景对象
  - `progress` (`Progress`) — 渲染进度追踪器
  - `params` (`SessionParams`) — 当前会话参数
  - `stats` (`Stats`) — 运行时统计
  - `profiler` (`Profiler`) — 性能分析器
  - `full_buffer_written_cb` (`std::function<void(string_view)>`) — 瓦片文件写入完成的回调

- **公开方法**:
  - `Session(params, scene_params)` — 构造函数：初始化任务调度器、创建设备和场景、配置路径追踪器、启动会话线程
  - `~Session()` — 析构函数：取消渲染、终止线程、按正确顺序销毁资源
  - `start()` — 发送信号启动渲染线程
  - `cancel(quick)` — 取消渲染。`quick=true` 时立即取消路径追踪而不等待缓冲区均匀采样
  - `draw()` — 触发路径追踪器的绘制操作
  - `wait()` — 等待渲染线程进入等待或结束状态
  - `ready_to_reset()` — 检查路径追踪器是否准备好重置
  - `reset(session_params, buffer_params)` — 延迟重置渲染参数和缓冲区（在下一次迭代开始时生效）
  - `set_pause(pause)` — 暂停/恢复渲染
  - `set_samples(samples)` — 动态更新采样数
  - `set_time_limit(time_limit)` — 动态更新时间限制
  - `set_output_driver(driver)` — 设置离线输出驱动
  - `set_display_driver(driver)` — 设置交互式显示驱动
  - `get_estimated_remaining_time()` — 估算剩余渲染时间
  - `get_progress()` — 获取渲染进度（0.0~1.0）
  - `device_free()` — 释放设备端资源
  - `collect_statistics(stats)` — 收集渲染统计信息
  - `process_full_buffer_from_disk(filename)` — 从磁盘读取完整帧文件并处理

- **保护成员（线程与同步）**:
  - `session_thread_` / `session_thread_cond_` / `session_thread_mutex_` — 会话线程及其同步原语
  - `session_thread_state_` — 线程状态枚举：`SESSION_THREAD_WAIT`、`SESSION_THREAD_RENDER`、`SESSION_THREAD_END`
  - `pause_` / `new_work_added_` / `pause_cond_` / `pause_mutex_` — 暂停控制
  - `tile_mutex_` / `buffers_mutex_` — 瓦片和缓冲区互斥锁
  - `delayed_reset_` (`DelayedReset`) — 延迟重置状态（包含 mutex、参数副本和标志）

- **保护成员（渲染管线）**:
  - `tile_manager_` (`TileManager`) — 瓦片管理器
  - `buffer_params_` (`BufferParams`) — 当前缓冲区参数
  - `render_scheduler_` (`RenderScheduler`) — 渲染调度器
  - `path_trace_` (`unique_ptr<PathTrace>`) — 路径追踪器实例

## 核心函数

### `thread_run()`
- **功能**: 会话线程主循环。在 WAIT/RENDER/END 三种状态间切换。渲染完成后回到 WAIT 状态等待下一次渲染信号。线程退出时刷新显示并销毁显示驱动。

### `thread_render()`
- **功能**: 执行一次完整渲染。启动性能分析器（CPU 设备时）、重置采样进度、运行主渲染循环、更新最终状态。

### `run_main_render_loop()`
- **功能**: 渲染主循环。每次迭代：获取下一个工作项 -> 检查取消 -> 检查暂停 -> 锁定缓冲区 -> 执行路径追踪 -> 更新状态。后台模式无工作时直接结束，交互模式等待新工作。

### `run_update_for_next_iteration()`
- **功能**: 为下一次渲染迭代准备工作。按顺序执行：延迟重置 -> 场景更新 -> 缓冲区更新 -> 降噪参数更新 -> 自适应采样更新 -> 路径引导更新 -> 获取渲染工作 -> 配置瓦片参数 -> 更新相机分辨率 -> 加载内核 -> 等待设备就绪。

### `run_wait_for_work()`
- **功能**: 在交互模式下等待渲染恢复或新工作到达。后台模式直接返回 false。等待期间跳过的时间不计入渲染时间。

### `delayed_reset_buffer_params()`
- **功能**: 执行延迟的缓冲区参数重置。将延迟重置参数应用到当前状态，更新曝光、阴影捕捉器、透明背景等配置，重新初始化瓦片调度。

### `update_buffers_for_params()`
- **功能**: 根据新参数更新缓冲区。配置渲染调度器、更新渲染通道、更新瓦片管理器、设置临时目录、重置进度。

### `update_scene()`
- **功能**: 更新场景状态。同步采样数到积分器、配置采样子集、设置采样计数通道、执行场景更新。

### `get_effective_tile_size()`
- **功能**: 计算有效瓦片大小。若未启用自动分块则返回整幅图像尺寸，否则通过 TileManager 计算兼容的瓦片大小，如果瓦片面积大于等于图像面积则不分块。

## 依赖关系
- **内部头文件**: `device/device.h`、`device/cpu/device.h`、`integrator/render_scheduler.h`、`integrator/path_trace.h`、`scene/shader.h`、`scene/stats.h`、`scene/scene.h`、`scene/background.h`、`scene/camera.h`、`scene/integrator.h`、`scene/film.h`、`session/buffers.h`、`session/tile.h`、`session/display_driver.h`、`session/output_driver.h`、`util/progress.h`、`util/stats.h`、`util/thread.h`、`util/unique_ptr.h`、`util/log.h`、`util/math.h`、`util/task.h`、`util/time.h`
- **外部库**: `<functional>`、`<cstring>`
- **被引用**: `session/tile.cpp`、`scene/scene.cpp`、`integrator/render_scheduler.cpp`、`hydra/session.cpp`、`hydra/render_pass.cpp`、`hydra/render_delegate.cpp`、`hydra/file_reader.h`、`app/cycles_standalone.cpp` 等共 9 个文件

## 实现细节 / 关键算法

### 延迟重置机制
`reset()` 不会立即执行重置，而是将参数存入 `delayed_reset_` 结构体并设置标志。实际重置在渲染循环的 `run_update_for_next_iteration()` 中执行。这避免了视口导航等频繁触发重置时从未完成任何渲染的问题。

### 线程状态机
会话线程在三个状态间转换：
- `SESSION_THREAD_WAIT` — 等待信号（空闲状态）
- `SESSION_THREAD_RENDER` — 正在渲染
- `SESSION_THREAD_END` — 线程终止

`start()` 将状态设为 RENDER 并通知线程，渲染完成后自动回到 WAIT。析构时设为 END 并 join 线程。

### 资源销毁顺序
析构函数中严格按以下顺序销毁资源：取消渲染 -> 终止线程 -> 销毁路径追踪器 -> 销毁场景 -> 销毁降噪设备 -> 销毁渲染设备 -> 停止任务调度器。路径追踪器必须在设备之前销毁，因为析构过程中可能需要释放设备内存。

### 渐进式渲染
通过 `resolution_divider` 实现渐进式分辨率提升。低分辨率渲染结果快速显示，然后逐步提高分辨率。相机参数按实际渲染分辨率更新。

## 关联文件
- `session/tile.h` — `TileManager` 管理瓦片分块与磁盘 I/O
- `session/buffers.h` — `BufferParams` 和 `RenderBuffers` 管理渲染数据
- `integrator/path_trace.h` — `PathTrace` 路径追踪器，执行实际渲染
- `integrator/render_scheduler.h` — `RenderScheduler` 渲染工作调度
- `session/display_driver.h` — 显示驱动接口
- `session/output_driver.h` — 输出驱动接口
- `scene/scene.h` — 场景管理
- `scene/integrator.h` — 积分器配置（采样、降噪、自适应采样参数）
