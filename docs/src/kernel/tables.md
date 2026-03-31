# tables.h - 内核预计算查找表数据

## 概述

`tables.h` 存储了 Cycles 内核在渲染过程中使用的预计算常量查找表数据。主要包括三类数据：黑体辐射色温-RGB 转换表、CIE 标准观察者颜色匹配函数表，以及 Sobol-Burley 低差异采样序列的方向向量表。这些数据均以 `ccl_inline_constant` 声明为编译期内联常量，直接嵌入到内核代码中，避免运行时内存查找开销。

## 类与结构体

无。本文件仅包含常量数组定义。

## 枚举与常量

### 黑体辐射查找表

- `blackbody_table_r[7][3]` (`float`) — 黑体辐射红色通道系数，7 个温度区间，每区间 3 个多项式系数
- `blackbody_table_g[7][3]` (`float`) — 黑体辐射绿色通道系数，结构同上
- `blackbody_table_b[7][4]` (`float`) — 黑体辐射蓝色通道系数，7 个温度区间，每区间 4 个多项式系数（蓝色通道使用更高阶的多项式拟合）

这些表用于根据色温（开尔文）快速计算对应的 RGB 颜色值，服务于 SVM 黑体节点。

### CIE 颜色匹配函数表

- `cie_color_match[81][3]` (`float`) — CIE 1931 标准观察者颜色匹配函数（x-bar, y-bar, z-bar），覆盖 380nm-780nm 可见光波段，以 5nm 为步长共 81 个采样点。用于光谱到 XYZ 色彩空间的转换。

### Sobol-Burley 采样方向向量

- `sobol_burley_table[4][32]` (`unsigned int`) — Sobol 序列前 4 个维度的方向向量，每维 32 个方向数（对应 32 位精度）。方向向量以位反转顺序存储，便于在运行时直接用于基于哈希的 Owen 置乱（scrambling），避免额外的位反转操作。

## 核心函数

无函数定义。

## 依赖关系

- **内部头文件**: `util/defines.h`（提供 `ccl_inline_constant` 宏定义）
- **被引用**:
  - `kernel/svm/math_util.h` — SVM 数学工具函数
  - `kernel/sample/sobol_burley.h` — Sobol-Burley 采样器实现
  - `kernel/device/optix/kernel.cu` — OptiX 内核
  - `kernel/device/gpu/kernel.h` — GPU 通用内核头文件

## 实现细节 / 关键算法

### 黑体辐射拟合

黑体辐射表采用**分段多项式拟合**方式近似普朗克黑体辐射定律。每个温度区间使用低阶多项式拟合色温到 RGB 分量的映射：
- 红色和绿色通道: 3 系数（二次多项式形式 `a + b*T + c*T^2` 或类似形式）
- 蓝色通道: 4 系数（三次多项式），因为蓝色在低温区域变化更剧烈，需要更高阶拟合

### Sobol-Burley 低差异序列

Sobol-Burley 采样器是 Cycles 的核心采样策略之一。方向向量仅需 4 个维度，因为更高维度通过填充（padding）机制实现。位反转存储是一种优化技巧——Owen 置乱算法本身需要位反转输入，预先存储反转数据避免了运行时的位反转计算。

### CIE 颜色匹配

81 组 CIE 颜色匹配数据是光谱渲染的基础，用于将单色光波长映射到人眼可感知的 XYZ 三刺激值，进而转换为渲染色彩空间。

## 关联文件

- `kernel/svm/blackbody.h` — 使用黑体辐射表实现色温到 RGB 的转换
- `kernel/sample/sobol_burley.h` — Sobol-Burley 采样器，直接使用 `sobol_burley_table`
- `kernel/svm/math_util.h` — 可能使用 CIE 颜色匹配数据
- `kernel/types.h` — 定义了采样模式枚举 `SAMPLING_PATTERN_SOBOL_BURLEY`
- `util/defines.h` — 提供 `ccl_inline_constant` 跨平台常量声明宏
