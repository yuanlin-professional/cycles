# noise.h - Perlin 噪声基础实现

## 概述
`noise.h` 实现了 1D 至 4D 的 Perlin 噪声函数，这是 Cycles 所有程序化噪声纹理的数学基础。代码改编自 Open Shading Language (OSL)，同时提供了标准 C++ 实现和 SSE/AVX 加速实现，在支持 SIMD 的 CPU 上可显著提升噪声计算性能。

文件还提供了安全版本的噪声函数（`snoise_*` 和 `noise_*`），对输入坐标进行周期性取模以防止浮点精度问题，输出范围分别为 [-1, 1]（有符号）和 [0, 1]（无符号）。

## 核心函数

### Perlin 噪声核心
- **`perlin_1d(x)`** - 1D Perlin 噪声
- **`perlin_2d(x, y)`** - 2D Perlin 噪声
- **`perlin_3d(x, y, z)`** - 3D Perlin 噪声
- **`perlin_4d(x, y, z, w)`** - 4D Perlin 噪声

### 辅助插值函数（标准实现）
- **`fade(t)`** - 5 次平滑阶梯函数：`t^3 * (t * (t*6 - 15) + 10)`，确保一阶和二阶导数连续
- **`bi_mix(v0, v1, v2, v3, x, y)`** - 双线性插值
- **`tri_mix(v0..v7, x, y, z)`** - 三线性插值
- **`quad_mix(v0..v15, x, y, z, w)`** - 四线性插值

### 梯度函数
- **`grad1(hash, x)`** - 1D 梯度：从哈希选择梯度大小(1-8)和方向
- **`grad2(hash, x, y)`** - 2D 梯度：8 个方向的梯度向量
- **`grad3(hash, x, y, z)`** - 3D 梯度：16 个方向的梯度向量（含 12 个边中点和 4 个对角修正）
- **`grad4(hash, x, y, z, w)`** - 4D 梯度：32 个方向的梯度向量

### 输出范围归一化
- **`noise_scale1/2/3/4(result)`** - 将原始 Perlin 噪声缩放到可预测的 [-1, 1] 范围，缩放因子（0.25, 0.6616, 0.9820, 0.8344）由 OSL 开发者通过实验确定

### 安全噪声接口
- **`snoise_1d/2d/3d/4d(p)`** - 有符号噪声，输出 [-1, 1]。对输入取模 100000.0 防止浮点溢出
- **`noise_1d/2d/3d/4d(p)`** - 无符号噪声，输出 [0, 1]，即 `0.5 * snoise + 0.5`

## 依赖关系
- **内部头文件**:
  - `util/hash.h` - 整数哈希函数 `hash_uint`, `hash_uint2/3/4`
  - `util/math.h` - 数学工具函数
  - `util/types.h` - 基础类型定义
- **被引用**:
  - `kernel/svm/fractal_noise.h` - 分形噪声算法使用 `snoise_1d/2d/3d/4d`
  - `kernel/svm/noisetex.h` - 噪声纹理节点的扰动计算

## 实现细节

### 三层架构设计
噪声实现根据编译环境分为三层：
1. **标准实现**（无 `__KERNEL_SSE__`）: 使用标量运算，适用于所有平台
2. **SSE 实现**（`__KERNEL_SSE__` 但无 `__KERNEL_AVX2__`）: 使用 SSE 指令并行计算 4 个梯度
3. **AVX 实现**（`__KERNEL_AVX2__`）: 使用 AVX 指令并行计算 8 个梯度

### SSE 并行化策略
对于 2D 噪声，SSE 实现一次计算所有 4 个格点的梯度值：
- 利用 `shuffle` 指令构造 4 个格点的坐标组合
- `hash_int4_2` 一次计算 4 个哈希值
- `grad` 函数一次计算 4 个梯度点积
- `bi_mix` 使用 SSE 指令完成双线性插值

对于 3D/4D，AVX 实现一次处理 8 个梯度（使用 `vfloat8`/`vint8` 类型），进一步提升吞吐量。

### 浮点精度保护
`snoise_*` 函数通过两种机制防止精度问题：
1. 对输入坐标取模 100000.0，防止大坐标值下的浮点表示退化
2. 当 `|p| >= 1000000.0` 时添加 0.5 的精度修正偏移

### `negate_if` / `negate_if_nth_bit` 优化
标准实现使用条件分支 `negate_if`，而 SSE/AVX 实现使用位操作 `negate_if_nth_bit`（异或符号位），消除了分支预测失败的开销。

## 关联文件
- `kernel/svm/fractal_noise.h` - 基于此噪声的分形叠加算法
- `kernel/svm/noisetex.h` - 噪声纹理 SVM 节点
- `kernel/svm/wave.h` - 波纹纹理（间接依赖，通过 fractal_noise）
- `util/hash.h` - 整数哈希函数
