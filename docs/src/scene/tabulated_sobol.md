# tabulated_sobol.h / tabulated_sobol.cpp - 表格化 Sobol 序列生成器

## 概述

本文件实现了 4D Owen 置乱 Sobol 序列的生成算法，基于 Helmer、Christensen 和 Kensler 的论文"Stochastic Generation of (t, s) Sample Sequences"。该序列用作 Cycles 路径追踪积分器的采样模式，提供低差异的准蒙特卡罗采样点集，确保渲染时样本分布均匀且收敛快速。

## 类与结构体

无类定义，仅包含一个核心函数。

## 核心函数

### `tabulated_sobol_generate_4D()`
- **签名**: `void tabulated_sobol_generate_4D(float4 points[], int size, int rng_seed)`
- **功能**: 生成指定数量的 4D Owen 置乱 Sobol 序列点
- **参数**:
  - `points` — 输出数组，存储生成的 float4 采样点（每个分量范围 [0,1)）
  - `size` — 需要生成的采样点数量
  - `rng_seed` — 随机数种子

## 依赖关系

- **内部头文件**: `util/types.h`
- **cpp 额外引用**: `util/hash.h`（`hash_hp_seeded_uint()`, `hash_hp_float()`）
- **被引用**: `scene/integrator.cpp`（积分器在设备更新时生成采样模式 LUT）

## 实现细节 / 关键算法

1. **Owen 置乱 Sobol**: 算法使用预定义的 XOR 值表 `xors[4][32]` 控制 4 个维度的层次划分顺序。不同的 XOR 值选择产生不同的置乱序列，当前选择确保生成 Owen 置乱的 Sobol 序列。
2. **层次细分**: 采用二分层次策略——从第一个随机点开始，反复将域细分为越来越细的分层（strata），每次细分时在未占用的分层中放置新点：
   - 外层循环 `log_N` 从 0 到 `log2(size)`，`N` 从 1 倍增到 `size`
   - 每次迭代对已有的 N 个点，通过 XOR 索引找到对应的已占用分层
   - 在对称的未占用分层（`occupied ^ 1`）中生成新点
3. **种子随机化**: 使用 `hash_hp_seeded_uint(rng_seed, 0x44605a73)` 对输入种子进行哈希处理，防止递增种子导致的相关性。
4. **高质量哈希**: 使用 `hash_hp_float()` 生成 [0,1) 范围的随机浮点数，在每个分层内提供均匀的抖动。
5. **内存布局**: 输出为紧凑的 `float4` 数组，4 个维度分别存储在 x/y/z/w 分量中，可直接用于路径追踪的多维采样。

## 关联文件

- `scene/integrator.cpp` — 积分器（调用此函数生成 `sample_pattern_lut` 并上传到设备）
- `util/hash.h` — 哈希函数（提供高质量随机数生成）
