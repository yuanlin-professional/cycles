# noisetex.h - 噪声纹理 SVM 节点

## 概述
`noisetex.h` 实现了 Cycles 的噪声纹理节点(`NODE_TEX_NOISE`)，这是 Blender 中最常用的程序化纹理之一。它将分形噪声算法封装为可在着色器图中使用的节点，支持 1D 至 4D 四种维度，五种噪声类型（fBm、多重分形、混合多重分形、脊状多重分形、异构地形），以及扰动(distortion)和颜色输出功能。

该文件定义了随机偏移生成函数，用于为不同的噪声通道创建伪随机种子，从而在同一坐标点生成不相关的多通道输出。

## 核心函数

### 随机偏移函数
- **`random_float_offset(seed)`** - 生成 [100, 200] 范围内的 1D 随机偏移
- **`random_float2_offset(seed)`** - 生成 2D 随机偏移
- **`random_float3_offset(seed)`** - 生成 3D 随机偏移
- **`random_float4_offset(seed)`** - 生成 4D 随机偏移

偏移范围选择在 [100, 200]，足够大以避免与原始坐标的噪声相关，又不至于大到引起浮点精度问题。使用浮点种子是为了与 OSL 的浮点哈希兼容。

### `noise_select<T>(p, detail, roughness, lacunarity, offset, gain, type, normalize)`
- **功能**: 模板分发函数，根据 `NodeNoiseType` 类型选择对应的分形噪声算法
- **支持类型**: `float`, `float2`, `float3`, `float4`
- **分发目标**:
  - `NODE_NOISE_MULTIFRACTAL` -> `noise_multi_fractal`
  - `NODE_NOISE_FBM` -> `noise_fbm`
  - `NODE_NOISE_HYBRID_MULTIFRACTAL` -> `noise_hybrid_multi_fractal`
  - `NODE_NOISE_RIDGED_MULTIFRACTAL` -> `noise_ridged_multi_fractal`
  - `NODE_NOISE_HETERO_TERRAIN` -> `noise_hetero_terrain`

### 纹理计算函数
- **`noise_texture_1d(co, ...)`** - 1D 噪声纹理，输入为单个浮点 `w`
- **`noise_texture_2d(co, ...)`** - 2D 噪声纹理，输入为 `float2`
- **`noise_texture_3d(co, ...)`** - 3D 噪声纹理，输入为 `float3`
- **`noise_texture_4d(co, ...)`** - 4D 噪声纹理，输入为 `float4`

每个函数：
1. 若 `distortion != 0`，先用噪声对坐标进行扰动
2. 调用 `noise_select` 计算主值
3. 若需要颜色输出，在偏移坐标上额外计算两个噪声通道，组成 RGB 颜色

### `svm_node_tex_noise(kg, stack, offsets1, offsets2, offsets3, node_offset)`
- **功能**: SVM 节点入口函数，处理参数解包、栈读写和维度分发
- **参数解包**: 从 3 个 `uint` 中解包 11 个栈偏移量（向量、W、缩放、细节、粗糙度、间隙度、偏移、增益、扰动、值输出、颜色输出）
- **默认值**: 从额外的 `uint4` 节点读取参数默认值
- **属性**: 从 `properties` 节点读取维度数(1-4)、噪声类型和归一化标志
- **约束**: `detail` 钳制到 [0, 15]，`roughness` 钳制到非负值

## 依赖关系
- **内部头文件**:
  - `kernel/svm/fractal_noise.h` - 五种分形噪声算法
  - `kernel/svm/util.h` - 栈操作和节点读取
- **被引用**:
  - `kernel/svm/svm.h` - 主解释器通过 `NODE_TEX_NOISE` case 分支调用

## 实现细节

### 扰动(Distortion)机制
扰动通过在原始坐标上添加噪声偏移实现：
```
p += snoise(p + random_offset(seed)) * distortion
```
每个坐标分量使用不同的随机偏移种子，确保各维度的扰动独立。扰动使用的是有符号噪声 `snoise_*`，产生双向偏移。

### 颜色输出生成
颜色输出的三个通道为：
- R: 主噪声值 `value`
- G: 在偏移坐标上的噪声（种子不同于主值）
- B: 在另一个偏移坐标上的噪声
通过 `color_is_needed` 标志（由 `stack_valid(color_stack_offset)` 决定）控制是否计算，避免不必要的开销。

### 缩放处理
坐标和 W 值在进入噪声函数前统一乘以 `scale`，实现全局频率缩放。

## 关联文件
- `kernel/svm/fractal_noise.h` - 分形噪声算法实现
- `kernel/svm/noise.h` - 基础 Perlin 噪声
- `kernel/svm/svm.h` - SVM 主解释器
- `kernel/svm/types.h` - `NodeNoiseType` 枚举
