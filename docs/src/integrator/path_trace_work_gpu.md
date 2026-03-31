# path_trace_work_gpu.h / path_trace_work_gpu.cpp - GPU 设备路径追踪工作实现

## 概述

`PathTraceWorkGPU` 是 `PathTraceWork` 的 GPU 设备实现，采用波前（wavefront）路径追踪架构，以图块为调度单位将工作提交到 GPU 设备队列。它管理 GPU 上的积分器状态（SoA 布局）、路径队列计数器、着色器排序、路径压缩以及图形互操作等核心 GPU 渲染机制。

与 CPU 的 megakernel 模式不同，GPU 实现将路径追踪拆分为多个内核（相交、着色、阴影等），通过队列调度实现高吞吐的波前并行执行。

## 类与结构体

### PathTraceWorkGPU

- **继承**: `PathTraceWork`
- **功能**: 在 GPU 设备上以波前模式执行路径追踪，管理积分器状态和内核调度
- **关键成员**:
  - `queue_` — GPU 设备队列（`DeviceQueue`），用于提交内核
  - `work_tile_scheduler_` — 图块调度器，将大图块拆分为小图块
  - `integrator_state_gpu_` — GPU 积分器状态（SoA 指针集合）
  - `integrator_state_soa_` — 积分器 SoA 数组的设备内存列表
  - `integrator_queue_counter_` — 内核队列计数器，追踪各内核的待处理路径数
  - `integrator_shader_sort_counter_` / `integrator_shader_sort_prefix_sum_` — 着色器排序相关缓冲区
  - `integrator_shader_sort_partition_key_offsets_` — 分区排序键偏移（支持局部原子排序）
  - `integrator_next_main_path_index_` / `integrator_next_shadow_path_index_` — 路径分裂索引
  - `queued_paths_` / `num_queued_paths_` — 排队路径数组
  - `work_tiles_` — 工作图块临时缓冲区
  - `display_rgba_half_` — 无图形互操作时的显示缓冲区
  - `device_graphics_interop_` — 图形互操作对象
  - `max_num_paths_` — 最大并发积分器状态数
  - `min_num_active_main_paths_` — 维持设备忙碌的最小活跃路径数
  - `max_active_main_path_index_` — 当前最大活跃路径索引
  - `num_sort_partitions_` — 排序分区数
- **关键方法**:
  - `alloc_work_memory()` — 分配所有 GPU 工作内存（SoA、队列、排序、路径分裂）
  - `init_execution()` — 初始化设备队列并拷贝积分器状态到常量内存
  - `render_samples(...)` — GPU 波前渲染主循环
  - `enqueue_work_tiles(finished)` — 从调度器获取图块并提交到 GPU
  - `enqueue_path_iteration()` — 选择队列最满的内核并调度执行
  - `enqueue_path_iteration(kernel, num_paths_limit)` — 调度指定内核执行
  - `compute_sorted_queued_paths(...)` — 按着色器排序路径以提高缓存一致性
  - `compact_main_paths()` / `compact_shadow_paths()` — 压缩路径数组以消除空洞
  - `copy_to_display(...)` — 支持图形互操作和回退的显示更新
  - `should_use_graphics_interop()` — 检查是否可使用图形互操作

## 核心函数

- **`render_samples()`**: GPU 波前渲染主循环：
  1. 设置 `WorkTileScheduler`：最大路径状态为 `max_num_paths_/8`（贪心调度策略）
  2. `enqueue_reset()` 重置积分器状态和计数器
  3. 主循环：
     - `enqueue_work_tiles()` 在路径数不足时添加新图块
     - `enqueue_path_iteration()` 选择队列最满的内核执行
     - 同步设备并更新队列计数器
     - 所有图块完成且无活跃路径时退出
  4. 计算平均占用率统计

- **`enqueue_path_iteration()`** (无参数版): 路径迭代调度核心：
  1. 统计所有内核的活跃路径总数
  2. 选择队列最满的内核（`get_most_queued_kernel`）
  3. 对创建阴影路径的内核，先检查阴影路径空间；空间不足时优先调度阴影内核
  4. 对 AO 内核限制路径数（每条创建两个阴影路径）
  5. 调度选定内核执行

- **`enqueue_path_iteration(kernel, limit)`** (指定内核版): 实际的内核调度：
  1. 决定工作大小和路径索引数组
  2. 对需要排序的内核调用 `compute_sorted_queued_paths()`（按着色器排序）
  3. 对其他内核使用 `compute_queued_paths()` 获取活跃路径数组
  4. 根据内核类型组装参数并通过 `queue_->enqueue()` 提交

- **`alloc_integrator_soa()`**: SoA 内存分配核心：
  1. 根据内核特性估算单状态大小
  2. 通过 `queue_->num_concurrent_states()` 确定最大路径数
  3. 使用宏模板展开 `state_template.h` 和 `shadow_state_template.h` 为每个字段分配设备内存
  4. 将设备指针写入 `integrator_state_gpu_` 结构

## 依赖关系

- **内部头文件**:
  - `kernel/integrator/state.h`
  - `device/graphics_interop.h`, `device/memory.h`, `device/queue.h`
  - `integrator/path_trace_work.h`, `integrator/work_tile_scheduler.h`
  - `util/vector.h`
- **实现文件额外依赖**:
  - `device/device.h`
  - `integrator/pass_accessor_gpu.h`, `integrator/path_trace_display.h`
  - `scene/scene.h`, `session/buffers.h`
  - `kernel/device/gpu/block_sizes.h`, `kernel/types.h`
  - `util/log.h`, `util/string.h`
- **被引用**: `integrator/path_trace_work.cpp`（在工厂方法中创建）

## 实现细节 / 关键算法

1. **波前（Wavefront）架构**: 路径追踪被拆分为多个内核——光线相交（intersect closest/shadow/subsurface/volume_stack/dedicated_light）和着色（shade background/light/shadow/surface/volume 等）。每次迭代选择队列最满的内核执行，保持 GPU 高占用率。

2. **SoA（Structure of Arrays）布局**: 积分器状态以 SoA 方式存储在 GPU 内存中，每个状态字段独立分配为一维数组。通过宏模板（`state_template.h`）自动展开，支持根据内核特性（`kernel_features`）条件分配，支持 packed 模式以减少内存占用。

3. **着色器排序**: 在执行着色内核前，按着色器 ID 对路径排序（`compute_sorted_queued_paths`），提高 GPU 线程束的缓存一致性和执行效率。支持两种排序实现：局部原子排序（`supports_local_atomic_sort`）和前缀和排序。

4. **路径压缩**: `compact_main_paths()` 和 `compact_shadow_paths()` 在路径终止后重组路径数组，消除空洞以减少后续内核的无效工作。压缩仅在活跃路径数明显小于最大索引时执行（最少 32 条路径）。

5. **阴影路径管理**: 阴影路径从主路径池的末端分配（`integrator_next_shadow_path_index_`）。当阴影路径空间不足时，优先调度阴影内核以释放空间；阴影路径不再活跃时压缩并重置索引。主路径数限制为最大路径数的一半，确保路径再生不会过早发生。

6. **图形互操作**: `copy_to_display()` 优先使用 GPU 图形互操作（避免 GPU->CPU->GPU 拷贝），回退到朴素拷贝（`copy_to_display_naive`）。互操作可用性通过 `should_use_graphics_interop()` 缓存检查。

7. **贪心图块调度**: `max_num_paths_/8` 作为单图块最大路径数，使图块更小，从而可以贪心地调度多个图块。这在 shadow catcher 等场景（路径数减半）时尤其重要，确保路径完成后能尽快补充新工作。

## 关联文件

- `integrator/path_trace_work.h` — 基类定义
- `integrator/path_trace_work_cpu.h` — CPU 对应实现
- `integrator/work_tile_scheduler.h` — GPU 图块调度器
- `device/queue.h` — GPU 设备队列接口
- `kernel/integrator/state_template.h` — 积分器状态 SoA 模板
