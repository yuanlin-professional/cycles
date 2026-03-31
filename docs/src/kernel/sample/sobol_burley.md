# sobol_burley.h - Sobol-Burley 扰乱采样器

## 概述
本文件实现了基于 Brent Burley 2020 年论文"Practical Hash-based Owen Scrambling"的洗牌（shuffled）Owen 扰乱 Sobol 采样器。与标准高维 Sobol 序列不同，该实现使用维度填充（padding）技术实现高维采样——仅使用少量基础 Sobol 维度（最多 4 维），通过不同的种子/洗牌来扩展到任意维度。这是 Cycles 中性能与质量兼顾的主力采样器之一。

## 类与结构体
无。

## 枚举与常量
函数内部使用多个固定哈希常量（如 `0xbff95bfe`、`0x635c77bd` 等），这些是精心选择的扰乱种子，用于确保不同维度集之间的去相关性。

## 核心函数

### sobol_burley()
- **签名**: `ccl_device_forceinline float sobol_burley(uint rev_bit_index, const uint dimension, const uint scramble_seed)`
- **功能**: 计算单个维度的 Owen 扰乱 Sobol 样本值。核心采样原语，被所有多维采样函数调用。
  - `rev_bit_index`：位反转后的样本索引
  - `dimension`：维度编号（0-3）
  - `scramble_seed`：Owen 扰乱种子
  - 维度 0 使用 Van der Corput 序列的快速路径（直接位反转），其他维度查表计算 Sobol 方向数并累积异或。最后应用 Owen 扰乱并转换为 `[0,1)` 浮点数。

### sobol_burley_sample_1D()
- **签名**: `ccl_device float sobol_burley_sample_1D(uint index, const uint dimension, uint seed, const uint shuffled_index_mask)`
- **功能**: 生成一维 Owen 扰乱洗牌 Sobol 样本。将维度编号混入种子以实现不同维度的去相关。先对索引进行 Owen 扰乱洗牌和掩码截断，再调用 `sobol_burley` 计算样本值。
  - `shuffled_index_mask`：序列长度掩码，必须为高位 1 低位 0 的形式（如 `0xffff0000`），其位反转值应不小于最大采样数。

### sobol_burley_sample_2D()
- **签名**: `ccl_device float2 sobol_burley_sample_2D(uint index, const uint dimension_set, uint seed, const uint shuffled_index_mask)`
- **功能**: 生成二维 Owen 扰乱洗牌 Sobol 样本。同一维度集内的两个维度具有分层性，不同维度集之间去相关。`dimension_set` 指定第几组二维采样（0 为前两维，1 为次两维，以此类推）。

### sobol_burley_sample_3D()
- **签名**: `ccl_device float3 sobol_burley_sample_3D(uint index, const uint dimension_set, uint seed, const uint shuffled_index_mask)`
- **功能**: 生成三维 Owen 扰乱洗牌 Sobol 样本。同一维度集内三个维度具有联合分层性。

### sobol_burley_sample_4D()
- **签名**: `ccl_device float4 sobol_burley_sample_4D(uint index, const uint dimension_set, uint seed, const uint shuffled_index_mask)`
- **功能**: 生成四维 Owen 扰乱洗牌 Sobol 样本。同一维度集内四个维度具有联合分层性。

## 依赖关系
- **内部头文件**:
  - `kernel/tables.h` - 提供 `sobol_burley_table` Sobol 方向数查找表
  - `kernel/sample/util.h` - 提供 `reversed_bit_owen` Owen 扰乱函数
  - `util/hash.h` - 提供 `hash_hp_uint` 哈希函数
- **被引用**:
  - `src/kernel/sample/pattern.h` - 采样模式调度器
  - `src/app/cycles_precompute.cpp` - 预计算工具

## 实现细节 / 关键算法

### Owen 扰乱 Sobol 序列
标准 Sobol 序列虽具有优秀的低差异性，但存在明显的结构化模式（尤其在低采样数时）。Owen 扰乱通过对 Sobol 样本进行分层随机置换来打破这些模式，同时保持低差异性。Burley 的贡献在于将 Owen 扰乱简化为一个高效的哈希操作（`reversed_bit_owen`），避免了传统 Owen 扰乱需要的树状结构。

### 维度填充（Padding）
传统 Sobol 序列为每个维度预计算方向数矩阵，高维采样需要大量存储。Burley 方法仅使用 4 个基础 Sobol 维度，通过改变洗牌/扰乱种子来"复用"这些维度，实现任意维度的采样。不同维度集之间通过不同的种子去相关，避免了维度间的伪相关。

### 洗牌与掩码优化
采样过程分两步：
1. **洗牌**：对位反转的索引应用 Owen 扰乱，实现样本顺序的随机排列
2. **掩码截断**：用 `shuffled_index_mask` 截断索引位数，在低采样数时提升性能（减少 Sobol 生成所需的迭代次数）

掩码必须是高位为 1、低位为 0 的形式（如 `0xFFFF0000`），因为洗牌是在位反转域进行的，掩码对应的有效位需要与最大采样数匹配。

### Van der Corput 快速路径
维度 0 的 Sobol 序列等价于 Van der Corput 序列（即简单的位反转），不需要查表计算方向数。由于维度填充机制会频繁复用维度 0，该快速路径带来了显著的性能提升。

### 去相关设计
每个维度/维度集的采样函数使用不同的固定常量与种子异或，确保：
- 1D 和 2D 采样即使用相同索引、维度、种子也不相关
- 不同维度集之间不相关
- 同一维度集内的维度保持分层

## 关联文件
- `src/kernel/sample/pattern.h` - 调用本文件的采样函数进行路径随机数生成
- `src/kernel/sample/util.h` - 提供 Owen 扰乱核心函数
- `src/kernel/sample/tabulated_sobol.h` - 另一种 Sobol 采样实现（预计算表格版）
- `src/kernel/tables.h` - Sobol 方向数表 `sobol_burley_table`
- `src/app/cycles_precompute.cpp` - 预计算工具，可能用于生成方向数表
