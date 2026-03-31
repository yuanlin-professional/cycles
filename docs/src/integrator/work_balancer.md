# work_balancer.h / work_balancer.cpp - 多设备负载均衡器

## 概述

`work_balancer` 模块提供多设备渲染场景下的工作负载均衡功能。通过分析各设备的实际渲染耗时，动态调整工作分配权重，使所有设备尽可能同时完成各自的渲染任务，从而最大化多设备并行效率。

该模块定义了 `WorkBalanceInfo` 结构体存储每设备的统计信息和权重，以及两个核心函数分别用于初始均分和基于统计的再平衡。

## 类与结构体

### WorkBalanceInfo

- **功能**: 存储单个设备的工作均衡信息
- **关键成员**:
  - `time_spent` — 执行工作所花费的时间（double）
  - `occupancy` — 设备执行工作时的平均占用率（0.0~1.0）
  - `weight` — 归一化权重，用于计算大图块中分配给该设备的比例

## 核心函数

- **`work_balance_do_initial(vector<WorkBalanceInfo>&)`**: 初始均分。在无统计信息时调用，将所有设备的权重设为 `1.0 / num_infos`。单设备时权重为 1.0，零设备时直接返回。

- **`work_balance_do_rebalance(vector<WorkBalanceInfo>&)`**: 基于统计的再平衡：
  1. 计算所有设备的总耗时和平均耗时
  2. 对每个设备，计算目标时间：`mix(info.time_spent, time_average, 1/N)`
  3. 新权重 = 旧权重 * 目标时间 / 实际耗时
  4. 归一化所有权重使其总和为 1
  5. 若所有设备的时间差异不超过 2%，视为已平衡，返回 false
  6. 否则更新权重并清零 `time_spent`，返回 true

## 依赖关系

- **内部头文件**:
  - `util/vector.h`
- **实现文件额外依赖**:
  - `util/math_base.h`
- **被引用**: `integrator/path_trace.h`（`PathTrace` 持有 `vector<WorkBalanceInfo>` 并调用均衡函数）

## 实现细节 / 关键算法

1. **渐进式均衡**: 再平衡使用 `mix()` 函数（线性插值），权重为 `1/N`，这意味着每次再平衡只调整误差的 `1/N`。这种渐进方式避免了因单次测量波动导致的权重剧烈变化，使均衡过程更加稳定。

2. **2% 阈值判定**: 若所有设备的目标时间与平均时间的偏差均不超过 2%（`std::fabs(1.0 - time_target / time_average) > 0.02`），认为设备间已足够平衡，跳过本次再平衡以避免不必要的缓冲区重新分片开销。

3. **权重语义**: 权重直接对应大图块按行切分时各设备分到的高度比例。例如两个设备权重分别为 0.6 和 0.4，则第一个设备渲染大图块 60% 的行，第二个设备渲染 40% 的行。

4. **与 PathTrace 的集成**: `PathTrace` 在 `rebalance()` 步骤中调用 `work_balance_do_rebalance()`，若返回 true 则重新计算各 `PathTraceWork` 的缓冲区切片参数。每次路径追踪后更新 `time_spent`。

## 关联文件

- `integrator/path_trace.h` — 使用者，持有均衡信息列表
- `integrator/render_scheduler.h` — 调度器决定何时触发再平衡
