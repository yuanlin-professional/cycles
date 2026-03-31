# fractal_noise.h - 分形噪声算法集合

## 概述
`fractal_noise.h` 实现了五种经典的分形噪声算法，它们都基于 Perlin 噪声的多频率叠加（八度叠加）。这些算法是程序化纹理生成的数学基础，被噪声纹理节点用于生成各种自然外观的图案，如云雾、地形、岩石表面等。

每种算法都提供了 1D(`float`)、2D(`float2`)、3D(`float3`)和 4D(`float4`) 四个维度的重载版本，通过 `detail`（细节层级/八度数）、`roughness`（粗糙度/振幅衰减）、`lacunarity`（间隙度/频率增长）等参数控制分形特性。

## 核心函数

### `noise_fbm(p, detail, roughness, lacunarity, normalize)`
- **算法**: 分形布朗运动(Fractional Brownian Motion, fBm)
- **原理**: 对 Perlin 噪声进行多个八度叠加，每个八度频率乘以 `lacunarity`，振幅乘以 `roughness`
- **特点**: 最基础的分形噪声，输出值在 0 附近波动。`normalize` 参数控制是否将结果归一化到 [0, 1]
- **应用**: 云雾、烟雾、通用程序化纹理

### `noise_multi_fractal(p, detail, roughness, lacunarity)`
- **算法**: 多重分形(Multifractal)
- **原理**: 使用乘法叠加（而非加法），`value *= (pwr * snoise(p) + 1.0)`
- **特点**: 产生更强的对比度和更复杂的分布，适合模拟多尺度特征
- **应用**: 山脉地形、岩石表面

### `noise_hetero_terrain(p, detail, roughness, lacunarity, offset)`
- **算法**: 异构地形(Heterogeneous Terrain)
- **原理**: 后续八度的贡献受前一八度输出值的加权调制，低处更平坦，高处更粗糙
- **特点**: 第一个八度不缩放，后续八度的增量为 `(snoise(p) + offset) * pwr * value`
- **应用**: 自然地形生成，具有海拔依赖的细节变化

### `noise_hybrid_multi_fractal(p, detail, roughness, lacunarity, offset, gain)`
- **算法**: 混合加法/乘法多重分形(Hybrid Multifractal)
- **原理**: 结合加法和乘法叠加，引入 `weight`（权重）概念，权重通过 `gain * signal` 更新
- **特点**: 当权重降至 0.001 以下时提前终止迭代，`offset` 和 `gain` 提供额外的艺术控制
- **应用**: 复杂地形，兼具平坦区域和锐利山脊

### `noise_ridged_multi_fractal(p, detail, roughness, lacunarity, offset, gain)`
- **算法**: 脊状多重分形(Ridged Multifractal)
- **原理**: 对噪声取绝对值后从 `offset` 中减去，再平方，产生尖锐的山脊效果。权重通过 `saturatef(signal * gain)` 限制在 [0, 1]
- **特点**: 产生尖锐的山脊和峡谷特征，`signal = (offset - |snoise(p)|)^2`
- **应用**: 山脊、裂缝、闪电形状的程序化图案

## 依赖关系
- **内部头文件**:
  - `kernel/svm/noise.h` - 提供基础 Perlin 噪声函数 `snoise_1d/2d/3d/4d`
- **被引用**:
  - `kernel/svm/noisetex.h` - 噪声纹理节点通过 `noise_select()` 调用这些函数
  - `kernel/svm/wave.h` - 波纹纹理使用 `noise_fbm` 生成扰动

## 实现细节

### 分数八度处理
所有算法都支持非整数的 `detail` 值。整数部分执行完整的八度循环，小数部分 (`rmd = detail - floor(detail)`) 通过线性插值（`mix`）混入最后一个部分八度的贡献，确保 detail 参数变化时输出平滑过渡而非阶跃。

### 归一化策略
fBm 函数的 `normalize` 参数启用时，输出通过 `0.5 * sum / maxamp + 0.5` 映射到 [0, 1] 范围，其中 `maxamp` 是所有八度振幅的累加和，确保无论 detail/roughness 如何变化，输出范围保持稳定。

### 性能标记
所有函数使用 `ccl_device_noinline` 标记，阻止编译器内联这些相对复杂的循环函数，减少 GPU 内核的寄存器压力和编译后代码体积。

## 关联文件
- `kernel/svm/noise.h` - 基础 Perlin 噪声（本文件的直接依赖）
- `kernel/svm/noisetex.h` - 噪声纹理节点（本文件的主要调用者）
- `kernel/svm/wave.h` - 波纹纹理（使用 fBm 生成扰动效果）
- `kernel/svm/types.h` - `NodeNoiseType` 枚举定义
