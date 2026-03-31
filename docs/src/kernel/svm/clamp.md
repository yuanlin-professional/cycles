# clamp.h - 值钳位节点

## 概述

`clamp.h` 实现了 Cycles SVM 中的钳位（Clamp）节点，对应 Blender 着色器编辑器中的"钳位（Clamp）"节点。该节点将输入值限制在指定的最小值和最大值范围内，支持两种钳位模式：最小-最大（Min-Max）模式和范围（Range）模式。

## 核心函数

### `svm_node_clamp`

```c
ccl_device_noinline int svm_node_clamp(KernelGlobals kg,
                                       ccl_private float *stack,
                                       const uint value_stack_offset,
                                       const uint parameters_stack_offsets,
                                       const uint result_stack_offset,
                                       int offset)
```

- **功能**: 钳位节点的 SVM 执行入口。
- **参数**:
  - `kg`: 内核全局数据，用于读取包含默认值的额外节点。
  - `stack`: SVM 栈指针。
  - `value_stack_offset`: 输入值的栈偏移量。
  - `parameters_stack_offsets`: 打包的参数偏移量（最小值偏移、最大值偏移、钳位类型）。
  - `result_stack_offset`: 结果的栈偏移量。
  - `offset`: 当前 SVM 指令偏移量。
- **返回值**: 更新后的 SVM 指令偏移量。

## 依赖关系

- **内部头文件**:
  - `kernel/svm/util.h` — 栈操作、节点解包和默认值加载工具
- **被引用**:
  - `kernel/svm/svm.h` — SVM 主调度器，在 `NODE_CLAMP` 指令分支中调用

## 实现细节 / 关键算法

### 两种钳位模式

1. **最小-最大模式（Min-Max, 默认）**: `result = clamp(value, min, max)`。
   - 始终使用 `min` 作为下限，`max` 作为上限。
   - 如果 `min > max`，结果行为未明确定义（取决于 `clamp` 函数行为）。

2. **范围模式（Range, `NODE_CLAMP_RANGE`）**: 自动处理 `min > max` 的情况。
   - 当 `min > max` 时，交换参数：`result = clamp(value, max, min)`。
   - 当 `min <= max` 时，正常钳位：`result = clamp(value, min, max)`。
   - 此模式对用户更友好，无论两个边界值的顺序如何都能正确工作。

### 默认值支持

通过 `stack_load_float_default()` 加载最小值和最大值，当栈偏移量无效（参数未连接）时使用 `read_node()` 读取的默认值（`defaults.x` 和 `defaults.y`）。

## 关联文件

- `kernel/svm/svm.h` — SVM 指令调度器
- `kernel/svm/map_range.h` — 范围映射节点（更复杂的值域变换）
- `kernel/svm/types.h` — `NODE_CLAMP_RANGE` 等常量定义
