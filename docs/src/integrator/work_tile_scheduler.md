# work_tile_scheduler.h / work_tile_scheduler.cpp - GPU 工作图块调度器

## 概述

`WorkTileScheduler` 负责将大图块（big tile）拆分为适合 GPU 设备队列处理的小工作图块（work tile），并按序分发给 GPU 执行。它的核心目标是保持 GPU 所有路径状态（path states）尽可能被利用，同时维持路径追踪的空间局部性以提高缓存效率。

该调度器主要被 `PathTraceWorkGPU` 使用，在每次 `render_samples()` 调用时重置并按需分发图块。

## 类与结构体

### WorkTileScheduler

- **功能**: 将图像空间和采样范围划分为小图块，按需分发给 GPU 内核
- **关键成员**:
  - `accelerated_rt_` — 是否有加速光线追踪支持（影响图块大小策略）
  - `max_num_path_states_` — 最大允许路径状态数（图块大小的限制因子）
  - `image_full_offset_px_` — 图像在全局缓冲区中的像素偏移
  - `image_size_px_` — 当前渲染图像尺寸（像素）
  - `offset_` / `stride_` — 缓冲区偏移和步幅
  - `scrambling_distance_` — 扰乱距离（影响图块大小策略）
  - `sample_start_` / `samples_num_` / `sample_offset_` — 采样范围参数
  - `tile_size_` — 计算得出的最优图块大小（`TileSize`）
  - `num_tiles_x_` / `num_tiles_y_` — X/Y 方向的图块数
  - `total_tiles_num_` — 图块总数
  - `num_tiles_per_sample_range_` — 覆盖完整采样范围所需的堆叠层数
  - `next_work_index_` — 下一个待分发的工作索引
  - `total_work_size_` — 总工作量（图块数 * 采样堆叠层数）
- **关键方法**:
  - `set_accelerated_rt(bool)` — 设置是否有加速 RT 支持
  - `set_max_num_path_states(int)` — 设置最大路径状态数
  - `reset(buffer_params, sample_start, samples_num, sample_offset, scrambling_distance)` — 用新参数重置调度器
  - `get_work(KernelWorkTile*, max_work_size)` — 获取下一个工作图块。返回 true 表示有工作，false 表示全部完成

## 核心函数

- **`reset()`**: 重置调度器，流程如下：
  1. 保存图像参数、采样参数
  2. 调用 `reset_scheduler_state()` 计算最优图块大小和图块网格

- **`reset_scheduler_state()`**: 核心初始化逻辑：
  1. 调用 `tile_calculate_best_size()` 计算最优图块大小
  2. 如果图块内路径状态数为 0，标记无工作
  3. 计算 X/Y 方向图块数（`divide_up`）
  4. 计算采样堆叠层数 `num_tiles_per_sample_range_`
  5. 计算总工作量 `total_work_size_`

- **`get_work()`**: 获取工作图块：
  1. 通过原子递增 `next_work_index_` 获取工作索引
  2. 解码索引为采样范围索引和空间图块索引（先采样后空间）
  3. 构造 `KernelWorkTile`：坐标、尺寸、采样范围
  4. 裁剪边界图块（不超过图像边界）
  5. 加上全局偏移
  6. 检查工作大小是否超过 `max_work_size` 限制
  7. 超限时回退索引以便后续重新获取

## 依赖关系

- **内部头文件**:
  - `integrator/tile.h`（`TileSize` 和 `tile_calculate_best_size()`）
  - `util/types_int2.h`
- **实现文件额外依赖**:
  - `device/queue.h`（`KernelWorkTile`）
  - `session/buffers.h`
  - `util/log.h`
- **被引用**: `integrator/path_trace_work_gpu.h`

## 实现细节 / 关键算法

1. **索引编码**: 工作索引编码为 `tile_index * num_tiles_per_sample_range_ + sample_range_index`，即先遍历采样范围再遍历空间图块。这使得同一空间位置的不同采样范围连续分发，有利于路径的时间局部性。

2. **采样堆叠**: 当单个图块的采样数小于总采样数时，同一空间图块会被多次调度（不同采样起点），称为"堆叠"。例如总采样 1024、图块采样 32，则同一空间图块调度 32 次。

3. **工作大小限制**: `get_work()` 的 `max_work_size` 参数允许调用方拒绝过大的图块。被拒绝的图块通过回退 `next_work_index_` 保留以供后续获取。这支持了 `PathTraceWorkGPU` 中的贪心多图块调度策略。

4. **边界裁剪**: 边缘图块的宽高会被裁剪到不超过图像边界，确保不会访问越界内存。

## 关联文件

- `integrator/tile.h` — 图块大小计算
- `integrator/path_trace_work_gpu.h` — 主要使用者
- `device/queue.h` — `KernelWorkTile` 结构定义
