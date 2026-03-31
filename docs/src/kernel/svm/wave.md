# wave.h - 波纹纹理 SVM 节点

## 概述
`wave.h` 实现了 Cycles 的波纹纹理节点(`NODE_TEX_WAVE`)，生成基于三角函数的条带和环状波纹图案。支持两种波纹类型（条带/环状）、多种方向模式以及三种波形轮廓（正弦/锯齿/三角），并可通过分形布朗运动(fBm)噪声对波纹进行扰动，产生自然的不规则波动效果。

该纹理结合了解析波形函数和程序化噪声，适合生成木纹、水波、布料褶皱等具有方向性周期的程序化图案。

## 核心函数

### `svm_wave(type, bands_dir, rings_dir, profile, p, distortion, detail, dscale, droughness, phase)`
- **功能**: 核心波纹计算函数
- **参数**:
  - `type`: 波纹类型（条带 `NODE_WAVE_BANDS` 或环状 `NODE_WAVE_RINGS`）
  - `bands_dir`: 条带方向（X/Y/Z/对角线）
  - `rings_dir`: 环状方向（X/Y/Z/球面）
  - `profile`: 波形轮廓（正弦/锯齿/三角）
  - `distortion`: 扰动强度
  - `detail`/`dscale`/`droughness`: fBm 噪声参数（细节、缩放、粗糙度）
  - `phase`: 相位偏移
- **返回**: `float`，[0, 1] 范围的波纹值

### `svm_node_tex_wave(kg, stack, node, offset)`
- **功能**: SVM 节点入口函数
- **输入**: 坐标(co)、缩放(scale)、扰动(distortion)、细节(detail)、扰动缩放(dscale)、扰动粗糙度(droughness)、相位(phase)
- **属性**: 波纹类型、条带方向、环状方向、波形轮廓
- **输出**:
  - **因子(Fac)**: 波纹值
  - **颜色**: `float3(f, f, f)` 灰度

## 依赖关系
- **内部头文件**:
  - `kernel/svm/fractal_noise.h` - 分形布朗运动(fBm)噪声，用于波纹扰动
  - `kernel/svm/util.h` - 栈操作和节点读取
- **被引用**:
  - `kernel/svm/svm.h` - 主解释器通过 `NODE_TEX_WAVE` case 调用

## 实现细节

### 条带模式(Bands)
基于单一轴或对角线方向生成平行条带：
- **X/Y/Z 方向**: `n = p.x/y/z * 20.0`
- **对角线方向**: `n = (p.x + p.y + p.z) * 10.0`

### 环状模式(Rings)
基于到某一轴或原点的距离生成同心环：
- **X/Y/Z 方向**: 将该轴分量置零后计算向量长度 `n = len(rp) * 20.0`
- **球面方向**: 直接使用到原点的距离

### 噪声扰动
当 `distortion != 0` 时，使用 fBm 噪声对波纹进行扰动：
```cpp
n += distortion * (noise_fbm(p * dscale, detail, droughness, 2.0, true) * 2.0 - 1.0)
```
fBm 噪声的 lacunarity 固定为 2.0，归一化后映射到 [-1, 1] 范围再乘以扰动强度。

### 三种波形轮廓
1. **正弦(Sin)**: `0.5 + 0.5 * sin(n - pi/2)` —— 平滑的正弦波，偏移 pi/2 使起点从 0 开始
2. **锯齿(Saw)**: `n/2pi - floor(n/2pi)` —— 线性上升后突然跳回，产生尖锐边缘
3. **三角(Tri)**: `|n/2pi - floor(n/2pi + 0.5)| * 2` —— 线性上升后线性下降，V 形波

### 精度保护
坐标在计算前进行微调 `(p + 0.000001) * 0.999999`，与 `checker.h` 类似，防止整数边界处的精度问题。

### 参数编码
波纹类型、方向和轮廓作为 RNA 属性（编译时常量）传递，在 SVM 编码中与栈偏移量一起打包在 `uint4` 中。尽管变量名为 `*_offset`，实际存储的是枚举值而非栈偏移。

## 关联文件
- `kernel/svm/fractal_noise.h` - fBm 噪声函数（扰动源）
- `kernel/svm/types.h` - `NodeWaveType`, `NodeWaveBandsDirection`, `NodeWaveRingsDirection`, `NodeWaveProfile` 枚举
- `kernel/svm/magic.h` - 另一种基于三角函数的纹理
- `kernel/svm/svm.h` - SVM 主解释器
