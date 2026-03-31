# lcg.h - 线性同余随机数生成器

## 概述
本文件实现了线性同余生成器（Linear Congruential Generator, LCG），提供快速的伪随机数生成功能。LCG 是路径追踪内核中最轻量的随机数来源，主要用于不需要高质量分层采样的场景（如体积散射中的随机游走、毛发着色器等），作为 Sobol 等高质量采样器的补充。模板化设计使其能适配 Metal 等多地址空间 GPU 后端。

## 类与结构体
无。

## 枚举与常量
| 常量 | 值 | 说明 |
|------|------|------|
| 乘数 a | `1103515245` | 经典 POSIX LCG 乘数 |
| 增量 c | `12345` | 经典 POSIX LCG 增量 |
| 模数 m | 2^32（隐式） | 利用 32 位无符号整数自然溢出实现取模 |

## 核心函数

### lcg_step_uint()
- **签名**: `template<class T> ccl_device uint lcg_step_uint(T rng)`
- **功能**: 执行一步 LCG 迭代，更新状态并返回 32 位无符号整数随机数。模板参数 `T` 为指向状态的指针类型，用于支持 Metal 不同地址空间。

### lcg_step_float()
- **签名**: `template<class T> ccl_device float lcg_step_float(T rng)`
- **功能**: 执行一步 LCG 迭代，将结果映射到 `[0, 1]` 浮点范围。通过乘以 `1.0f / 0xFFFFFFFF` 进行归一化。

### lcg_step_float3()
- **签名**: `template<class T> ccl_device float3 lcg_step_float3(T rng)`
- **功能**: 连续生成三个浮点随机数，组合为 `float3` 向量。显式按 x、y、z 顺序求值，确保跨平台结果一致。

### lcg_init()
- **签名**: `ccl_device uint lcg_init(const uint seed)`
- **功能**: 用给定种子初始化 LCG 状态。先执行一步迭代以避免直接使用种子值作为首个输出（消除种子相关性）。

### lcg_state_init()
- **签名**: `ccl_device_inline uint lcg_state_init(const uint rng_hash, const uint rng_offset, const uint sample, const uint scramble)`
- **功能**: 基于像素哈希、偏移量、采样索引和扰动值组合生成 LCG 初始状态。内部调用 `hash_uint3` 进行混合，为每个采样上下文产生独立的随机序列。

## 依赖关系
- **内部头文件**: `util/hash.h`（提供 `hash_uint3` 哈希函数）
- **被引用**:
  - `src/kernel/util/texture_3d.h`
  - `src/kernel/light/light.h`
  - `src/kernel/integrator/shade_volume.h`
  - `src/kernel/geom/volume.h`
  - `src/kernel/integrator/intersect_dedicated_light.h`
  - `src/kernel/geom/geom_intersect.h`
  - `src/kernel/device/gpu/kernel.h`
  - `src/kernel/device/cpu/bvh.h`
  - `src/kernel/closure/bsdf_principled_hair_huang.h`

## 实现细节 / 关键算法
- **经典 POSIX LCG**: 采用公式 `state = 1103515245 * state + 12345`，模数为 2^32（通过无符号整数溢出隐式实现）。该参数组合源自 POSIX `rand()` 标准，周期为 2^32。
- **模板化地址空间**: 所有步进函数使用模板参数 `T` 而非固定指针类型。这是为了兼容 Metal Shading Language 中的多种地址空间（`device`、`thread`、`threadgroup` 等），使同一份代码可在不同 GPU 后端编译。
- **浮点归一化**: `lcg_step_float` 将 32 位整数除以 `0xFFFFFFFF` 映射到 `[0, 1]` 闭区间，而非更常见的 `[0, 1)` 半开区间。
- **用途定位**: LCG 的分层质量较低，不适合主要采样维度。在 Cycles 中它主要用于辅助随机化（如体积散射方向采样、毛发 BSDF 内部采样等），这些场景对样本关联性不敏感。

## 关联文件
- `src/kernel/sample/pattern.h` - 主采样模式调度器，使用 Sobol/蓝噪声作为主采样序列
- `src/kernel/sample/sobol_burley.h` - 高质量 Sobol-Burley 采样器
- `src/kernel/sample/tabulated_sobol.h` - 预计算表格 Sobol 采样器
- `src/util/hash.h` - 提供 `hash_uint3` 等哈希工具函数
