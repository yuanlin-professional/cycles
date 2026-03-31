# kernel/sample - 采样系统

## 概述

`kernel/sample/` 实现了 Cycles 路径追踪渲染器的采样子系统，负责生成高质量的准随机数序列和执行各类采样操作。该模块提供了多种采样模式（Sobol-Burley、表格化 Sobol、蓝噪声）、多重重要性采样（MIS）权重计算、均匀分布映射（球面、半球面、圆盘等），以及线性同余随机数生成器。

采样质量直接影响渲染结果的收敛速度和噪点分布。Cycles 使用低差异序列（Sobol 序列）配合 Owen 扰乱来生成每像素、每弹射、每维度的采样值，确保样本在高维空间中的良好分层特性。

## 目录结构

| 文件名 | 类型 | 说明 |
|--------|------|------|
| `pattern.h` | 头文件 | 采样模式调度入口：根据积分器配置选择 Sobol-Burley、表格化 Sobol 或蓝噪声序列，提供 `path_rng_1D`/`path_rng_2D` 等采样函数 |
| `sobol_burley.h` | 头文件 | Sobol-Burley 采样器：基于 Brent Burley 2020 年论文的 Owen 扰乱 Sobol 序列实现，使用维度填充（padding）达到高维采样 |
| `tabulated_sobol.h` | 头文件 | 表格化 Sobol 采样器：预计算的 Sobol 序列查找表，支持混洗（shuffling）和 Cranley-Patterson 旋转去相关 |
| `lcg.h` | 头文件 | 线性同余生成器（LCG）：快速伪随机数生成，用于次表面散射等需要局部随机数的场景 |
| `mis.h` | 头文件 | 多重重要性采样（MIS）工具：平衡启发式（balance heuristic）、幂启发式（power heuristic）和最大值启发式（max heuristic）权重计算 |
| `mapping.h` | 头文件 | 采样映射工具：均匀随机数到几何分布的映射函数（圆盘、球面、半球面、三角形等采样） |
| `util.h` | 头文件 | 采样辅助工具：Owen 扰乱哈希函数（base-2 和 base-4）、嵌套均匀扰乱（`nested_uniform_scramble`） |

## 核心类与数据结构

### 采样模式枚举
- **`SAMPLING_PATTERN_SOBOL_BURLEY`** - Sobol-Burley 序列（默认），每像素独立序列
- **`SAMPLING_PATTERN_TABULATED_SOBOL`** - 表格化 Sobol 序列，使用预计算查找表
- **`SAMPLING_PATTERN_BLUE_NOISE_PURE`** - 纯蓝噪声序列，所有像素共享单一长序列
- **`SAMPLING_PATTERN_BLUE_NOISE_FIRST`** - 首样本蓝噪声，第一个样本使用蓝噪声分布（优化视口导航的 1 SPP 质量）

### PathTraceDimension 枚举（定义于 kernel/types.h）
为路径追踪中每个弹射步骤的不同采样需求分配唯一维度编号：
- **初始弹射**: `PRNG_FILTER`（像素滤波）、`PRNG_LENS_TIME`（透镜/时间）
- **表面着色**: `PRNG_TERMINATE`（俄罗斯轮盘）、`PRNG_LIGHT`（光源选择）、`PRNG_SURFACE_BSDF`（BSDF 方向）
- **体积着色**: `PRNG_VOLUME_PHASE`（相位函数）、`PRNG_VOLUME_SCATTER_DISTANCE`（散射距离）
- **次表面散射**: `PRNG_SUBSURFACE_BSDF`、`PRNG_SUBSURFACE_SCATTER_DISTANCE`

## 内核函数入口

| 函数 | 文件 | 说明 |
|------|------|------|
| `path_rng_1D()` | `pattern.h` | 生成路径追踪 1D 采样值（自动选择采样模式） |
| `path_rng_2D()` | `pattern.h` | 生成路径追踪 2D 采样值 |
| `path_rng_3D()` | `pattern.h` | 生成路径追踪 3D 采样值 |
| `blue_noise_indexing()` | `pattern.h` | 蓝噪声索引计算，根据采样模式将像素和样本映射到序列索引 |
| `sobol_burley_sample_1D()` | `sobol_burley.h` | Sobol-Burley 单维度采样 |
| `sobol_burley_sample_2D()` | `sobol_burley.h` | Sobol-Burley 二维采样 |
| `sobol_burley_sample_3D()` | `sobol_burley.h` | Sobol-Burley 三维采样 |
| `sobol_burley_sample_4D()` | `sobol_burley.h` | Sobol-Burley 四维采样 |
| `tabulated_sobol_sample_1D()` | `tabulated_sobol.h` | 表格化 Sobol 单维度采样 |
| `tabulated_sobol_sample_2D()` | `tabulated_sobol.h` | 表格化 Sobol 二维采样 |
| `tabulated_sobol_sample_3D()` | `tabulated_sobol.h` | 表格化 Sobol 三维采样 |
| `lcg_step_float()` | `lcg.h` | LCG 单步浮点随机数 |
| `lcg_step_float3()` | `lcg.h` | LCG 单步三维浮点随机数 |
| `lcg_state_init()` | `lcg.h` | LCG 状态初始化 |
| `balance_heuristic()` | `mis.h` | MIS 平衡启发式权重 |
| `power_heuristic()` | `mis.h` | MIS 幂启发式权重 |
| `sample_uniform_disk()` | `mapping.h` | 同心映射的均匀圆盘采样 |
| `make_orthonormals_tangent()` | `mapping.h` | 从法线和切线构建正交坐标系 |

## GPU 兼容性

- 所有采样函数使用 `ccl_device_forceinline` 确保在 GPU 上内联展开
- `sobol_burley_table` 使用 `ccl_inline_constant` 声明，放置在 GPU 常量内存中
- LCG 使用模板参数处理 Metal 的多地址空间
- 表格化 Sobol 的查找表数据通过 `kernel_data_fetch(sample_pattern_lut, ...)` 从全局内存读取
- 调试模式 `__DEBUG_CORRELATION__` 仅在 CPU 单线程下可用

## 依赖关系

### 上游依赖（本模块依赖）
- `util/hash.h` - 哈希函数（`hash_wang_seeded_uint`、`hash_wang_seeded_float`、`hash_shuffle_uint`、`hash_uint3`）
- `util/math.h` - 基础数学函数
- `util/projection.h` - 球面坐标转换
- `kernel/globals.h` - 内核全局数据（`kernel_data.integrator` 采样配置）
- `kernel/tables.h` - Sobol-Burley 方向向量常量表

### 下游依赖（依赖本模块）
- `kernel/integrator/` - 路径积分器在每个弹射步骤调用 `path_rng_*` 生成随机数
- `kernel/camera/` - 相机模块使用采样值生成透镜和像素抖动
- `kernel/light/` - 光源采样使用随机数选择光源和方向
- `kernel/closure/` - 闭包采样使用随机数选择散射方向
- `kernel/bvh/` - 局部 BVH 遍历使用 LCG 做蓄水池采样

## 关键算法与实现细节

### Sobol-Burley 采样器（sobol_burley.h）
基于 Brent Burley 2020 年 JCGT 论文 "Practical Hash-based Owen Scrambling" 实现。核心思想：
1. 使用 4 维 Sobol 方向向量表（存储在 `kernel/tables.h` 中，比特反转存储以优化 Owen 扰乱）
2. 通过维度填充（padding）将 4 维 Sobol 序列扩展到任意维度
3. 使用快速哈希实现 Owen 扰乱（`reversed_bit_owen`），避免了传统 Owen 扰乱的高开销
4. 每像素使用不同的种子实现像素间去相关

### Owen 扰乱哈希（util.h）
实现了 base-2 和 base-4 的 Owen 扰乱哈希：
- **base-2 Owen**: 基于 psychopath.io 博客的高质量哈希，操作反转比特序列
- **base-4 Owen**: 扩展到 base-4，提供更好的分层质量，同样基于 psychopath.io 方案
- `nested_uniform_scramble()` 包装函数处理比特反转前后的转换

### 蓝噪声索引（pattern.h）
三种蓝噪声模式的索引计算策略：
- **纯蓝噪声**: 所有像素共享单一全局序列，每像素分段使用（`sample + pixel_index * sequence_length`）
- **首样本蓝噪声**: 第一个样本使用独立的蓝噪声序列（种子 `0x0cd0519f`），后续样本使用 N-1 长度的序列。用于视口导航时 1 SPP 的蓝噪声质量
- 索引掩码优化：Sobol-Burley 模式使用 `sobol_index_mask` 避免不必要的高位计算

### 多重重要性采样（mis.h）
实现了 Veach 论文中的 MIS 权重计算，适配自 Open Shading Language：
- **平衡启发式**: `w = a / (a + b)`，无偏但方差较大
- **幂启发式**: `w = a^2 / (a^2 + b^2)`，接近最优，是 Cycles 默认使用的方法
- **最大值启发式**: 仅用于调试，选择概率最高的策略

## 参见

- `src/kernel/tables.h` - Sobol-Burley 方向向量表和其他常量查找表
- `src/kernel/types.h` - `PathTraceDimension` 采样维度枚举
- `src/kernel/integrator/` - 积分器中的采样调用点
- `src/util/hash.h` - 底层哈希函数实现
