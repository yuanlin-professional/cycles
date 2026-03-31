# guiding.h - 路径引导参数定义

## 概述

本文件定义了 Cycles 渲染器路径引导（Path Guiding）功能的参数结构体 `GuidingParams`。路径引导是一种先进的渲染优化技术，通过学习光照分布来引导光线采样方向，提高复杂照明场景的收敛速度。该结构体封装了控制引导场创建和重建的全部关键参数。

## 类与结构体

### GuidingParams

- **继承**: 无（POD 结构体）
- **功能**: 存储路径引导功能的配置参数，用于判断是否需要重建引导场。
- **关键成员**:
  - `use` — 是否启用路径引导（默认 `false`）
  - `use_surface_guiding` — 是否启用表面引导（默认 `false`）
  - `use_volume_guiding` — 是否启用体积引导（默认 `false`）
  - `type` — 引导分布类型（默认 `GUIDING_TYPE_PARALLAX_AWARE_VMM`，视差感知 VMM 分布）
  - `sampling_type` — 方向采样类型（默认 `GUIDING_DIRECTIONAL_SAMPLING_TYPE_PRODUCT_MIS`，乘积 MIS 采样）
  - `roughness_threshold` — 粗糙度阈值（默认 `0.05f`），低于此值的表面不使用引导
  - `training_samples` — 训练采样数（默认 `128`），用于构建引导场的初始采样数
  - `deterministic` — 是否使用确定性模式（默认 `false`）
- **关键方法**:
  - `modified(other)` — 比较两个参数集是否有差异，任何参数变化都返回 `true`，用于判断是否需要重建引导场

## 核心函数

无独立函数。`modified()` 方法是唯一的行为方法，通过逐字段比较判断参数是否发生变化。

## 依赖关系

- **内部头文件**:
  - `kernel/types.h` — 引导分布类型枚举（`GuidingDistributionType`、`GuidingDirectionalSamplingType`）
- **被引用**:
  - `scene/integrator.h` — 积分器配置中包含引导参数
  - `integrator/path_trace.h` — 路径追踪器使用引导参数

## 实现细节 / 关键算法

### 引导分布类型

`GUIDING_TYPE_PARALLAX_AWARE_VMM`（视差感知的 von Mises-Fisher 混合模型）是默认的引导分布类型，它通过考虑观察点位置的变化来提高引导场的空间适应性。

### 采样策略

`GUIDING_DIRECTIONAL_SAMPLING_TYPE_PRODUCT_MIS` 使用乘积多重重要性采样（Product MIS），将引导分布与 BSDF 采样结合，实现更高效的采样策略。

### 变化检测

`modified()` 方法用于渲染管线中检测参数是否变化。如果任何参数发生改变，路径追踪器需要重建引导场，这是一个计算开销较大的操作。因此精确的变化检测对避免不必要的重建至关重要。

## 关联文件

- `src/scene/integrator.h` — 积分器配置（包含 `GuidingParams` 成员）
- `src/integrator/path_trace.h` — 路径追踪器（使用引导参数）
- `src/kernel/types.h` — 引导类型枚举定义
