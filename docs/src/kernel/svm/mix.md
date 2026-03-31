# mix.h - 混合节点 SVM 执行入口

## 概述

`mix.h` 是 Cycles SVM 中混合（Mix）节点的执行入口文件，为颜色混合、浮点数混合、向量混合以及非均匀向量混合提供了完整的 SVM 节点实现。该文件从 SVM 栈读取输入，调用 `color_util.h` 中的底层混合函数，并处理混合因子钳位和结果钳位等选项。

## 核心函数

### `svm_node_mix`

```c
ccl_device_noinline int svm_node_mix(KernelGlobals kg,
                                     ccl_private float *stack,
                                     const uint fac_offset,
                                     const uint c1_offset,
                                     const uint c2_offset,
                                     int offset)
```

- **功能**: 旧版颜色混合节点入口（兼容用途）。
- **流程**: 从栈加载混合因子 `fac`、颜色 `c1` 和 `c2`，读取额外节点获取混合类型和输出偏移，调用 `svm_mix_clamped_factor()`（因子自动钳位至 [0,1]），将结果存入栈。

### `svm_node_mix_color`

```c
ccl_device_noinline void svm_node_mix_color(ccl_private float *stack,
                                            const uint options,
                                            const uint input_offset,
                                            const uint result_offset)
```

- **功能**: 颜色混合节点入口，支持可选的因子钳位和结果钳位。
- **选项参数**（从 `options` 解包）:
  - `use_clamp`: 是否钳位混合因子到 [0,1]
  - `blend_type`: 混合类型（对应 `NodeMix` 枚举）
  - `use_clamp_result`: 是否钳位最终结果到 [0,1]
- **流程**: 解包选项和输入偏移，加载因子和颜色，调用 `svm_mix()` 执行混合。

### `svm_node_mix_float`

```c
ccl_device_noinline void svm_node_mix_float(ccl_private float *stack,
                                            const uint use_clamp,
                                            const uint input_offset,
                                            const uint result_offset)
```

- **功能**: 浮点数线性混合节点。
- **公式**: `result = a * (1 - t) + b * t`（标准线性插值）。

### `svm_node_mix_vector`

```c
ccl_device_noinline void svm_node_mix_vector(ccl_private float *stack,
                                             const uint input_offset,
                                             const uint result_offset)
```

- **功能**: 向量均匀混合节点，使用单个标量因子对三分量向量进行线性插值。
- **公式**: `result = a * (1 - t) + b * t`

### `svm_node_mix_vector_non_uniform`

```c
ccl_device_noinline void svm_node_mix_vector_non_uniform(ccl_private float *stack,
                                                         const uint input_offset,
                                                         const uint result_offset)
```

- **功能**: 向量非均匀混合节点，使用三分量向量作为混合因子，每个分量独立控制对应轴的混合比例。
- **公式**: `result = a * (1 - t) + b * t`，其中 `t` 为 float3。

## 依赖关系

- **内部头文件**:
  - `kernel/svm/color_util.h` — 提供 `svm_mix()`、`svm_mix_clamped_factor()` 等混合运算函数
  - `kernel/svm/util.h` — 栈操作和节点解包工具
- **被引用**:
  - `kernel/svm/svm.h` — SVM 主调度器，在 `NODE_MIX`、`NODE_MIX_COLOR`、`NODE_MIX_FLOAT`、`NODE_MIX_VECTOR`、`NODE_MIX_VECTOR_NON_UNIFORM` 指令分支中调用

## 实现细节

1. **新旧接口并存**: `svm_node_mix` 是旧版接口（需要额外读取一个 SVM 节点），`svm_node_mix_color` 是新版接口，将选项直接编码在指令参数中，避免额外的内存读取。
2. **因子钳位**: 颜色混合支持可选的因子钳位（`use_clamp`），浮点数和向量混合也同样支持，通过 `saturatef()` / `saturate()` 实现。
3. **结果钳位**: 仅颜色混合模式（`svm_node_mix_color`）支持结果钳位选项，使用 `saturate()` 将 RGB 各通道限制在 [0,1]。

## 关联文件

- `kernel/svm/color_util.h` — 颜色混合核心算法
- `kernel/svm/svm.h` — SVM 指令调度器
- `kernel/svm/types.h` — `NodeMix` 枚举定义
