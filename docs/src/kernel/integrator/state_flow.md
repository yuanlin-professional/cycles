# state_flow.h - 积分器内核调度控制流

## 概述

`state_flow.h` 提供了积分器内核之间的控制流工具函数，管理路径在不同内核之间的转移、初始化和终止。CPU 和 GPU 使用不同的实现策略：CPU 上直接修改状态中的排队标记，GPU 上通过原子操作管理内核队列计数器和排序键。该文件是积分器波前(wavefront)调度架构的基础。

## 核心函数

### 路径终止检查
- **`integrator_path_is_terminated`**: 检查主路径是否已终止（`queued_kernel == 0`）。
- **`integrator_shadow_path_is_terminated`**: 检查阴影路径是否已终止。

### 光学深度写入
- **`write_optical_depth`**: 在路径终止时将体积主要透射光学深度写入渲染缓冲区。

### 主路径调度（GPU 版本）
- **`integrator_path_init`**: 初始化新路径，原子递增目标内核队列计数。
- **`integrator_path_next`**: 从当前内核转移到下一个内核，原子递减当前计数、递增下一计数。
- **`integrator_path_terminate`**: 终止路径，写入光学深度，原子递减当前内核计数。
- **`integrator_path_init_sorted`**: 初始化路径并设置排序键（结合状态索引的局部性和着色器的一致性）。
- **`integrator_path_next_sorted`**: 带排序键的内核转移。

### 阴影路径调度（GPU 版本）
- **`integrator_shadow_path_init`**: 初始化阴影路径，从全局索引池原子分配阴影状态。
- **`integrator_shadow_path_next`**: 阴影路径的内核转移。
- **`integrator_shadow_path_terminate`**: 终止阴影路径。

### CPU 版本
CPU 版本的所有调度函数仅修改 `queued_kernel` 字段，不涉及原子操作或排序。阴影路径初始化根据 `is_ao` 参数选择 `state->shadow` 或 `state->ao`。

## 依赖关系

- **内部头文件**:
  - `kernel/globals.h` - 全局内核数据
  - `kernel/types.h` - 类型定义
  - `kernel/film/write.h` - 胶片写入
  - `kernel/integrator/state.h` - 状态数据结构
  - `util/atomic.h` - 原子操作（仅 GPU）
- **被引用**: `shadow_catcher.h`, `shadow_linking.h`, `intersect_shadow.h`, GPU/OptiX/CPU 内核调度文件

## 实现细节 / 关键算法

1. **波前调度模型**: GPU 上使用波前(wavefront)路径追踪，每个内核处理一批路径后，通过 `integrator_path_next` 将路径放入下一个内核的队列。`queue_counter` 数组记录每个内核待处理的路径数，主机端根据计数决定启动哪些内核。

2. **排序键设计**: `INTEGRATOR_SORT_KEY` 宏将排序键定义为 `key + max_shaders * (state / partition_divisor)`，同时考虑着色器一致性（相同材质的路径连续处理）和空间局部性（相邻像素的路径分在同一分区），优化 GPU 缓存命中率。

3. **GPU 阴影路径索引池**: 阴影路径通过 `next_shadow_path_index` 原子分配，使多个主路径可以同时分支出阴影光线而不冲突。CPU 上则直接使用主路径结构体内嵌的阴影状态。

4. **光学深度追踪**: 路径终止时调用 `write_optical_depth`，将累计的体积主要光学深度写入专用通道，用于后续降噪或体积渲染分析。

## 关联文件

- `state.h` - 提供状态数据结构和访问宏
- `path_state.h` - 更高层的路径状态操作（弹射计数、随机数等）
- `shadow_catcher.h` - 使用路径终止检查函数
- `shadow_linking.h` - 使用内核调度函数
