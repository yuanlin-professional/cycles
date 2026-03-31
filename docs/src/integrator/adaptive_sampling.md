# adaptive_sampling.h / adaptive_sampling.cpp - 自适应采样控制

## 概述

本文件实现了 Cycles 渲染器的自适应采样逻辑控制类 `AdaptiveSampling`。自适应采样允许渲染器根据像素的收敛程度动态决定每个像素的采样数量，已收敛的像素提前停止采样，从而显著减少渲染时间。本类负责采样对齐和过滤时机的判断，确保所有设备（包括多 GPU 场景）在完全相同的采样点执行过滤操作。

## 类与结构体

### AdaptiveSampling

- **继承**: 无
- **功能**: 管理自适应采样的步长对齐和过滤时机判断，确保多设备间的一致性。
- **关键成员**:
  - `use` — 是否启用自适应采样（默认 `false`）
  - `adaptive_step` — 自适应采样步长（必须为 2 的幂），每隔此步数执行一次收敛性过滤
  - `min_samples` — 最小采样数，在此之前不执行自适应过滤
  - `threshold` — 收敛阈值（本类不直接使用，由内核端使用）
- **关键方法**:
  - `align_samples(start_sample, num_samples)` — 对齐采样数，返回到下一个过滤点为止最多可渲染的采样数
  - `need_filter(sample)` — 判断给定采样索引是否需要执行自适应过滤

## 核心函数

### align_samples()

将请求的采样数量对齐到最近的过滤点。这确保了渲染器不会跳过任何过滤点，从而保证所有设备在同一采样索引上执行过滤检查。

算法：
1. 如果自适应采样未启用，直接返回 `num_samples`
2. 计算第一个过滤采样索引：`(min_samples + 1) | (adaptive_step - 1)`
3. 如果请求范围完全在第一个过滤点之前，直接返回 `num_samples`
4. 否则计算到下一个过滤点的距离，返回较小值

### need_filter()

判断是否需要在给定采样索引处执行过滤：
1. 自适应采样未启用时返回 `false`
2. 采样数不超过 `min_samples` 时返回 `false`
3. 使用位运算 `(sample & (adaptive_step - 1)) == (adaptive_step - 1)` 判断是否为步长的倍数边界

## 依赖关系

- **内部头文件**:
  - `util/math.h` — 数学工具函数（`min`、`max`）
- **被引用**:
  - `integrator/render_scheduler.h` — 渲染调度器使用自适应采样参数
  - `scene/integrator.h` — 积分器场景配置
  - `test/integrator_adaptive_sampling_test.cpp` — 单元测试

## 实现细节 / 关键算法

### 位运算优化

`adaptive_step` 要求为 2 的幂次方，这使得过滤点判断可以用高效的位运算实现：

- `sample & (adaptive_step - 1)` 等价于 `sample % adaptive_step`
- `(adaptive_step - 1)` 作为位掩码，当低位全为 1 时表示到达步长边界

### 第一个过滤点计算

`(min_samples + 1) | (adaptive_step - 1)` 计算出满足两个条件的最小采样索引：
1. 大于 `min_samples`（确保最少采样数）
2. 是 `adaptive_step` 的对齐边界

### 多设备一致性

`align_samples()` 的设计确保无论设备性能差异如何，所有设备都会在完全相同的采样点执行过滤操作，从而保证最终结果的确定性。

## 关联文件

- `src/integrator/render_scheduler.h` — 使用 `AdaptiveSampling` 调度渲染工作
- `src/scene/integrator.h` — 积分器参数定义（包含自适应采样配置）
- `src/test/integrator_adaptive_sampling_test.cpp` — 对应的单元测试
