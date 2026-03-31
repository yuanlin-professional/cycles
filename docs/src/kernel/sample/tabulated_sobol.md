# tabulated_sobol.h - 预计算表格 Sobol 采样器

## 概述
本文件实现了基于预计算查找表（LUT）的 Sobol 采样器。与 `sobol_burley.h` 的实时计算方式不同，该采样器将 Sobol 序列预先计算并存储在查找表中，运行时通过索引直接读取样本值。通过维度洗牌、索引扰乱和可选的 Cranley-Patterson 旋转，在保持采样质量的同时提供灵活的扰乱距离控制。该采样器特别适合需要精细控制像素间采样相关性的场景。

## 类与结构体
无。

## 枚举与常量
引用的外部常量：

| 常量 | 说明 |
|------|------|
| `NUM_TAB_SOBOL_PATTERNS` | 预计算 Sobol 模式的总数 |
| `NUM_TAB_SOBOL_DIMENSIONS` | 每个样本的 Sobol 维度数（用于 LUT 索引步长） |

## 核心函数

### tabulated_sobol_shuffled_sample_index()
- **签名**: `ccl_device uint tabulated_sobol_shuffled_sample_index(KernelGlobals kg, uint sample, const uint dimension, const uint seed)`
- **功能**: 计算经洗牌后的样本在查找表中的实际索引。执行两层随机化：
  1. 使用 `hash_shuffle_uint` 对模式顺序进行洗牌（在 `NUM_TAB_SOBOL_PATTERNS` 个模式中随机选取一个）
  2. 使用 `nested_uniform_scramble` 对样本索引进行 Owen 扰乱（仅在当前模式范围内洗牌，避免过早重复使用模式）

  最终索引 = `(pattern_i * sample_count + sample) % (sample_count * NUM_TAB_SOBOL_PATTERNS)`。

### tabulated_sobol_sample_1D()
- **签名**: `ccl_device float tabulated_sobol_sample_1D(KernelGlobals kg, const uint sample, const uint rng_hash, const uint dimension)`
- **功能**: 从预计算表中获取一维 Sobol 样本。当扰乱距离 `scrambling_distance < 1.0` 时，使用全局种子替代像素哈希作为序列种子（使所有像素共享相同序列），并应用受限的 Cranley-Patterson 旋转进行微调。

### tabulated_sobol_sample_2D()
- **签名**: `ccl_device float2 tabulated_sobol_sample_2D(KernelGlobals kg, const uint sample, const uint rng_hash, const uint dimension)`
- **功能**: 从预计算表中获取二维 Sobol 样本。读取连续两个维度的值，并在需要时分别对 x、y 分量应用 Cranley-Patterson 旋转。

### tabulated_sobol_sample_3D()
- **签名**: `ccl_device float3 tabulated_sobol_sample_3D(KernelGlobals kg, const uint sample, const uint rng_hash, const uint dimension)`
- **功能**: 从预计算表中获取三维 Sobol 样本。读取连续三个维度的值。

### tabulated_sobol_sample_4D()
- **签名**: `ccl_device float4 tabulated_sobol_sample_4D(KernelGlobals kg, const uint sample, const uint rng_hash, const uint dimension)`
- **功能**: 从预计算表中获取四维 Sobol 样本。读取连续四个维度的值。

## 依赖关系
- **内部头文件**:
  - `kernel/globals.h` - 内核全局数据结构，提供 `kernel_data` 和 `kernel_data_fetch`
  - `kernel/sample/util.h` - 提供 `nested_uniform_scramble` Owen 扰乱函数
  - `util/hash.h` - 提供 `hash_shuffle_uint`、`hash_wang_seeded_uint`、`hash_wang_seeded_float` 等哈希函数
- **被引用**:
  - `src/kernel/sample/pattern.h` - 采样模式调度器

## 实现细节 / 关键算法

### 预计算查找表结构
样本数据存储在 `sample_pattern_lut` 纹理/缓冲区中。每个样本占 `NUM_TAB_SOBOL_DIMENSIONS` 个浮点值（对应多个维度），总共有 `sample_count * NUM_TAB_SOBOL_PATTERNS` 个样本条目。索引方式为 `index * NUM_TAB_SOBOL_DIMENSIONS + dim_offset`。

### 双层洗牌策略
为最大化有限预计算模式的利用效率，采样器实施两层随机化：
1. **模式选择洗牌**：使用 `hash_shuffle_uint` 随机选取一个预计算模式。不同维度选取不同模式，增加维度间的去相关性。
2. **样本索引扰乱**：在选定模式内部，使用 `nested_uniform_scramble`（base-2 Owen 扰乱）对样本顺序进行扰乱。关键约束是扰乱仅在当前模式的 `sample_count` 范围内进行（通过 `sample_mask` 确保），避免跨模式边界洗牌导致过早重复。

### Cranley-Patterson 旋转
当 `scrambling_distance < 1.0` 时，采样器进入"受限扰乱"模式：
1. 所有像素使用相同的全局种子，共享同一序列顺序
2. 对每个像素施加基于其 `rng_hash` 的 Cranley-Patterson 旋转（即对样本值加上一个随机偏移，然后取模 1）
3. 旋转幅度由 `scrambling_distance` 控制：0 表示所有像素完全相同（最大相关性），1 表示完全独立

这种机制使用户可以平滑地在"相邻像素完全相关"（有利于去噪器）和"完全独立"（传统蒙特卡洛）之间插值。

### 与 Sobol-Burley 的对比
| 特性 | Tabulated Sobol | Sobol-Burley |
|------|----------------|--------------|
| 采样值计算 | 查表 | 实时计算 |
| 内存占用 | 较高（LUT） | 较低（方向数表） |
| 扰乱距离控制 | 支持 | 不支持 |
| 维度扩展 | 预计算维度数固定 | 通过填充扩展到任意维度 |

## 关联文件
- `src/kernel/sample/pattern.h` - 调用本文件的采样函数
- `src/kernel/sample/sobol_burley.h` - 另一种 Sobol 采样实现（实时计算版）
- `src/kernel/sample/util.h` - 提供 Owen 扰乱工具函数
- `src/kernel/globals.h` - 提供 `sample_pattern_lut` 查找表访问接口
