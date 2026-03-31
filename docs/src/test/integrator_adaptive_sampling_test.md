# integrator_adaptive_sampling_test.cpp - 自适应采样算法测试

## 概述

此文件测试 Cycles 渲染器中自适应采样（Adaptive Sampling）模块的核心功能。自适应采样通过在已收敛的像素区域提前停止采样来加速渲染过程。测试验证了采样对齐（`align_samples`）、采样调度（`schedule_samples`）和过滤判断（`need_filter`）等关键函数的正确性。

## 测试用例

### TEST(AdaptiveSampling, schedule_samples)
- **功能**: 验证采样调度的正确性。遍历多种 `sample` 和 `num_samples` 组合，确保经过 `align_samples` 对齐后的采样数量在完成渲染时恰好处于需要进行过滤检测的采样点上（即 `need_filter` 返回 `true`）。
- **参数配置**: `min_samples=0`, `adaptive_step=4`
- **验证逻辑**: 对每一组参数，`sample + num_samples_aligned - 1` 必须是一个过滤点

### TEST(AdaptiveSampling, align_samples)
- **功能**: 详细测试 `align_samples` 函数在各种边界条件下的行为。该函数将请求的采样数对齐到下一个过滤点，以确保不会跳过任何过滤检测。
- **参数配置**: `min_samples=11`, `adaptive_step=4`（过滤发生在第 15、19、23、27... 个采样处）
- **验证场景**:
  - 从第 0 个采样开始，请求少于最小采样数的情况（原样返回）
  - 从第 0 个采样开始，请求多于第一个过滤点的情况（限制到第一个过滤点）
  - 从非零起始采样开始的对齐
  - 起始采样恰好在过滤点上的情况
  - 起始采样在过滤点之后的情况
  - 确保返回值永远不超过请求的采样数

### TEST(AdaptiveSampling, need_filter)
- **功能**: 验证 `need_filter` 函数正确标记需要进行收敛检测的采样编号。
- **参数配置**: `min_samples=11`, `adaptive_step=4`
- **验证逻辑**: 遍历 0-59 号采样，收集所有 `need_filter` 返回 `true` 的采样编号，与预期列表 `{15, 19, 23, 27, 31, 35, 39, 43, 47, 51, 55, 59}` 进行比对

## 依赖关系
- **被测源文件**: `integrator/adaptive_sampling.h`
- **测试框架**: Google Test (GTest)
- **辅助依赖**: `util/vector.h`

## 关联文件
- `src/integrator/adaptive_sampling.h` - AdaptiveSampling 结构体定义
- `src/integrator/adaptive_sampling.cpp` - AdaptiveSampling 实现
