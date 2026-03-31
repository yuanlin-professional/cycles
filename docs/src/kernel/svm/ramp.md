# ramp.h - 颜色渐变与曲线映射节点

## 概述

`ramp.h` 实现了 Cycles SVM 中的颜色渐变（Color Ramp / RGB Ramp）节点和曲线映射（Curves）节点，对应 Blender 着色器编辑器中的"颜色渐变（ColorRamp）"、"RGB 曲线（RGB Curves）"和"浮点曲线（Float Curve）"节点。这些节点通过查找预计算的颜色/数值表来实现任意的颜色映射和值变换。

## 核心函数

### `fetch_float`

```c
ccl_device_inline float fetch_float(KernelGlobals kg, const int offset)
```

- **功能**: 从 SVM 节点数组中读取单个浮点值，用于浮点渐变表的查找。

### `float_ramp_lookup`

```c
ccl_device_inline float float_ramp_lookup(KernelGlobals kg, const int offset,
                                          float f, bool interpolate,
                                          bool extrapolate, const int table_size)
```

- **功能**: 浮点渐变表查找，支持插值和外推。
- **参数**:
  - `offset`: 表在 SVM 节点数组中的起始偏移。
  - `f`: 查找位置，标准范围 [0, 1]。
  - `interpolate`: 是否在采样点之间线性插值。
  - `extrapolate`: 是否在范围外外推。
  - `table_size`: 查找表大小。
- **外推**: 超出 [0,1] 范围时，使用表边缘两点的斜率进行线性外推。

### `rgb_ramp_lookup`

```c
ccl_device_inline float4 rgb_ramp_lookup(KernelGlobals kg, const int offset,
                                         float f, bool interpolate,
                                         bool extrapolate, const int table_size)
```

- **功能**: RGBA 颜色渐变表查找，逻辑与 `float_ramp_lookup` 相同，但操作 float4 数据（含 Alpha 通道）。

### `svm_node_rgb_ramp`

```c
ccl_device_noinline int svm_node_rgb_ramp(KernelGlobals kg,
                                          ccl_private float *stack,
                                          const uint4 node, int offset)
```

- **功能**: 颜色渐变（ColorRamp）节点的 SVM 执行入口。
- **输入**: 因子 `fac`（查找位置）。
- **输出**: 颜色（float3）和 Alpha（float），分别写入对应的栈位置。
- **流程**: 读取表大小，调用 `rgb_ramp_lookup()` 查找颜色，分别存储 RGB 和 Alpha。

### `svm_node_curves`

```c
ccl_device_noinline int svm_node_curves(KernelGlobals kg,
                                        ccl_private float *stack,
                                        const uint4 node, int offset)
```

- **功能**: RGB 曲线节点的 SVM 执行入口。
- **输入**: 因子 `fac`、输入颜色。
- **流程**: 将输入颜色的 R、G、B 通道分别归一化到曲线的定义域 [min_x, max_x]，独立查找各通道对应的曲线值，然后与原始颜色按因子混合。

### `svm_node_curve`

```c
ccl_device_noinline int svm_node_curve(KernelGlobals kg,
                                       ccl_private float *stack,
                                       const uint4 node, int offset)
```

- **功能**: 浮点曲线节点的 SVM 执行入口。
- **输入**: 因子 `fac`、输入浮点值。
- **流程**: 将输入值归一化到曲线定义域，查找对应曲线值，与原始值按因子混合。

## 依赖关系

- **内部头文件**:
  - `kernel/svm/util.h` — 栈操作、`fetch_node_float()` 和节点解包工具
- **被引用**:
  - `kernel/svm/svm.h` — SVM 主调度器，在 `NODE_RGB_RAMP`、`NODE_CURVES` 和 `NODE_CURVE` 指令分支中调用

## 实现细节 / 关键算法

### 查找表存储

颜色渐变和曲线数据以查找表形式内联存储在 SVM 字节码流中。表大小通过额外节点读取（`read_node(kg, &offset).x`），然后偏移量跳过整个表（`offset += table_size`）。

### 插值与外推

- **插值**: 查找位置落在两个采样点之间时，使用线性插值 `(1-t)*a + t*b`。
- **外推**: 超出 [0,1] 范围时，使用边缘斜率进行线性外推。外推使用表的前两个点或后两个点计算斜率 `dy`，然后 `result = edge_value + dy * distance * (table_size - 1)`。
- **NaN 防护**: 整数索引使用 `clamp(float_to_int(f), 0, table_size - 1)` 确保在 NaN 输入时不会越界。

### RGB 曲线的独立通道处理

RGB 曲线节点对 R、G、B 通道使用同一张查找表，但分别取 `result.x`、`result.y`、`result.z`，这意味着表的每个条目存储了三个独立的曲线值（红色曲线、绿色曲线、蓝色曲线）。

## 关联文件

- `kernel/svm/ramp_util.h` — CPU 端渐变查找的工具版本（使用指针访问而非 SVM 节点）
- `kernel/svm/svm.h` — SVM 指令调度器
- `scene/shader_nodes.cpp` — 场景层节点编译，将渐变/曲线数据编码到 SVM 字节码
