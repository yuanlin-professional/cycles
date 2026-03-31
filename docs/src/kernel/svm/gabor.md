# gabor.h - Gabor 噪声纹理 SVM 节点

## 概述
`gabor.h` 实现了 Cycles 的 Gabor 噪声纹理节点(`NODE_TEX_GABOR`)，这是一种基于 Gabor 核函数的高质量程序化噪声。实现基于 Lagae 等人 (2009) 的稀疏 Gabor 卷积论文，并融合了 Tavernier 等人 (2019) 的快速归一化改进和 Tricard 等人 (2019) 的相位噪声理论。

Gabor 噪声相比 Perlin 噪声具有更好的频谱控制能力，支持各向异性和各向同性两种模式，可输出值、相位和强度三种结果，特别适合创建方向性纹理和波纹效果。

## 核心函数

### 2D Gabor 噪声

#### `compute_2d_gabor_kernel(position, frequency, orientation)`
- **功能**: 计算 2D Gabor 核函数值（复数形式）
- **算法**: 高斯包络 * Hann 窗口 * 余弦调制
  - 高斯包络: `exp(-pi * |p|^2)`
  - Hann 窗口: `0.5 + 0.5 * cos(pi * |p|^2)`（抑制截断不连续性）
  - 调制: 极坐标形式的复数 phasor
- **返回**: `float2` 复数，实部为余弦分量，虚部为正弦分量（使用正弦以确保零均值）

#### `compute_2d_gabor_standard_deviation()`
- **功能**: 计算 2D Gabor 噪声的近似标准差
- **推导**: 对 Gabor 核平方进行二重积分，乘以脉冲密度和伯努利分布二阶矩
- **近似值**: `sqrt(IMPULSES_COUNT * 0.5 * 0.25)` (高频极限下积分 -> 0.25)

#### `compute_2d_gabor_noise_cell(cell, position, frequency, isotropy, base_orientation)`
- **功能**: 计算单个格子内的 Gabor 噪声，叠加 `IMPULSES_COUNT`(8) 个脉冲
- **随机化**: 每个脉冲的方向、位置、权重使用不同哈希种子
- **权重**: 使用伯努利分布(+1/-1)而非均匀分布，如 Tavernier 论文建议
- **优化**: 当脉冲中心到采样点距离 >= 1 时提前跳过

#### `compute_2d_gabor_noise(coordinates, frequency, isotropy, base_orientation)`
- **功能**: 在 3x3 邻域格子中累加 Gabor 噪声

### 3D Gabor 噪声

#### `compute_3d_gabor_kernel(position, frequency, orientation)`
- **功能**: 3D Gabor 核函数，方向使用三维向量而非角度

#### `compute_3d_gabor_standard_deviation()`
- **功能**: 3D 版本标准差，积分分母为 `4 * sqrt(2)` 而非 4

#### `compute_3d_orientation(orientation, isotropy, seed)`
- **功能**: 在球坐标系中计算 3D 方向，在各向异性基础方向和随机方向间按 `isotropy` 插值

#### `compute_3d_gabor_noise_cell` / `compute_3d_gabor_noise`
- **功能**: 3D 版本的格子噪声和邻域累加（3x3x3 = 27 个格子）

### SVM 节点入口
#### `svm_node_tex_gabor(kg, stack, type, stack_offsets_1, stack_offsets_2, offset)`
- **功能**: SVM 节点主函数
- **输入**: 坐标、缩放、频率、各向异性度、2D 方向角/3D 方向向量
- **输出**:
  - **值(Value)**: `phasor.y / (6 * sigma) * 0.5 + 0.5` 映射到 [0, 1]
  - **相位(Phase)**: `atan2(phasor.y, phasor.x)` 归一化到 [0, 1]
  - **强度(Intensity)**: `|phasor| / (6 * sigma)`

## 依赖关系
- **内部头文件**:
  - `kernel/svm/util.h` - 栈操作和节点读取
  - `util/hash.h` - 哈希函数（`hash_float3_to_float`, `hash_float4_to_float3` 等）
- **被引用**:
  - `kernel/svm/svm.h` - 主解释器通过 `NODE_TEX_GABOR` case 调用

## 实现细节

### 常量 IMPULSES_COUNT = 8
每个格子固定 8 个脉冲，这是分层泊松点采样策略。原始论文建议从泊松分布采样脉冲数，但 Tavernier 证明固定数量配合伯努利权重效果更好。

### Phasor 噪声形式
使用复数 phasor（极坐标到直角坐标转换）而非直接计算 Gabor 值。这允许在叠加后提取：
- **相位**: `atan2(Im, Re)` —— 复数辐角
- **强度**: `|phasor|` = `sqrt(Re^2 + Im^2)` —— 复数模

### 各向异性控制
`isotropy` 参数为 `1 - anisotropy`：
- `isotropy = 0`（完全各向异性）: 所有脉冲使用基础方向
- `isotropy = 1`（完全各向同性）: 方向完全随机
- 中间值: 在基础方向上添加 `isotropy * random_angle` 的随机偏转

### Hann 窗口的必要性
原始 Gabor 论文声称高斯截断产生的伪影可以忽略，但实际在凹凸映射(bump mapping)求导时会产生明显的不连续性。Hann 窗口是 C1 连续的，有效消除了这些伪影。

### 归一化因子
经验确定的 `6 * sigma` 归一化因子将输出映射到大致 [-1, 1] 范围，然后再线性映射到 [0, 1]。

## 关联文件
- `kernel/svm/types.h` - `NodeGaborType` 枚举（2D / 3D）
- `kernel/svm/svm.h` - SVM 主解释器
- `util/hash.h` - 随机数生成
