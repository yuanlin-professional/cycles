# tile.h / tile.cpp - 图块大小计算

## 概述

`tile` 模块提供图块大小（`TileSize`）的定义和最优图块大小的计算算法。其核心功能是根据图像尺寸、采样数和最大路径状态数，计算出既能充分利用 GPU 路径状态又能保持空间局部性的最优图块尺寸。

该模块是 GPU 波前路径追踪调度的基础，`WorkTileScheduler` 在每次重置时调用 `tile_calculate_best_size()` 来确定调度粒度。

## 类与结构体

### TileSize

- **功能**: 描述工作图块的三维尺寸（宽度、高度、采样数）
- **关键成员**:
  - `width` — 图块宽度（像素）
  - `height` — 图块高度（像素）
  - `num_samples` — 图块包含的采样数
- **关键方法**:
  - 构造函数 `TileSize(width, height, num_samples)`
  - `operator==` / `operator!=` — 相等比较
  - `operator<<(ostream&, TileSize&)` — 流输出（用于日志）

## 核心函数

- **`tile_calculate_best_size(accel_rt, image_size, num_samples, max_num_path_states, scrambling_distance)`**: 最优图块大小计算，核心算法：

  1. **特殊情况**: 若 `max_num_path_states == 1`，返回 1x1x1 避免计算精度问题
  2. **完全适配**: 若整张图像的所有采样可放入路径状态，返回整张图像大小
  3. **扰乱距离模式**（`scrambling_distance < 0.9 && accel_rt`）: 偏好大图块，宽度尽可能为图像宽度，高度由剩余路径状态决定
  4. **通常模式**: 偏好小图块 + 多采样:
     - 计算每采样可用路径状态数 `max_states / num_samples`
     - 图块宽高取其平方根并向下取整为 2 的幂
     - 图块采样数取 `sqrt(num_samples/2)` 并向上取整为 2 的幂
     - 确保总路径数不超过最大值

- **`round_down_to_power_of_two(x)`**: 向下取整到最近的 2 的幂

- **`round_up_to_power_of_two(x)`**: 向上取整到最近的 2 的幂

## 依赖关系

- **内部头文件**:
  - `util/types_int2.h`
- **实现文件额外依赖**:
  - `util/log.h`, `util/math_base.h`
- **被引用**: `integrator/work_tile_scheduler.h`, `integrator/work_tile_scheduler.cpp`, `test/integrator_tile_test.cpp`

## 实现细节 / 关键算法

1. **2 的幂对齐**: 图块宽高和采样数均取 2 的幂值，这确保了：
   - 图块能整数倍地填入路径状态池（减少浪费）
   - 采样范围的均匀划分（如 `[32x38次, 8]` 优于 `[1024, 200]`），有利于贪心调度

2. **空间局部性优先**: 通常模式下偏好较小的空间图块配合较多采样数。这使得同一图块内的路径在空间上更紧凑，提高光线遍历和纹理访问的缓存命中率。

3. **扰乱距离特殊处理**: 当 `scrambling_distance < 0.9` 且有加速 RT 时，偏好大图块（尽可能覆盖整张图像宽度）。这是因为低扰乱距离场景中空间一致性更重要。

4. **采样数启发式**: `sqrt(num_samples/2)` 的启发式旨在使采样划分更均匀。例如 1024 采样被划分为 32 组，每组 32 采样，而非 1 组 1024 或 1024 组 1。这允许 GPU 在渲染过程中更早地开始贪心调度额外图块。

## 关联文件

- `integrator/work_tile_scheduler.h` — 图块调度器，调用 `tile_calculate_best_size()`
- `integrator/path_trace_work_gpu.h` — GPU 工作实现
