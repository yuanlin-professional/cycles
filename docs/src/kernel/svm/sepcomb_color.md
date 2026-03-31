# sepcomb_color.h - 颜色分离与合并节点的着色器虚拟机实现

## 概述

本文件实现了着色器虚拟机（SVM）中的颜色合并（Combine Color）和颜色分离（Separate Color）节点。这两个节点支持在不同颜色空间（如 RGB、HSV、HSL）之间进行颜色分量的合并与拆分操作，是着色器编辑器中颜色操控的基础工具。

## 核心函数

### `svm_node_combine_color`

```c
ccl_device_noinline void svm_node_combine_color(
    ccl_private float *stack,
    const uint color_type,
    const uint inputs_stack_offsets,
    const uint result_stack_offset)
```

**功能**：将三个独立的颜色分量合并为一个颜色值。

**参数**：
- `stack`：SVM 栈指针
- `color_type`：颜色空间类型（`NodeCombSepColorType` 枚举，如 RGB、HSV、HSL）
- `inputs_stack_offsets`：打包的三个输入分量栈偏移量
- `result_stack_offset`：输出颜色的栈偏移量

**执行流程**：
1. 使用 `svm_unpack_node_uchar3` 解包三个输入偏移量
2. 从栈中加载三个浮点分量（R/G/B 或 H/S/V 等）
3. 调用 `svm_combine_color` 将分量组合并转换为 RGB 颜色
4. 将结果颜色存入栈

### `svm_node_separate_color`

```c
ccl_device_noinline void svm_node_separate_color(
    ccl_private float *stack,
    const uint color_type,
    const uint input_stack_offset,
    const uint results_stack_offsets)
```

**功能**：将一个颜色值分离为三个独立分量。

**参数**：
- `stack`：SVM 栈指针
- `color_type`：目标颜色空间类型
- `input_stack_offset`：输入颜色的栈偏移量
- `results_stack_offsets`：打包的三个输出分量栈偏移量

**执行流程**：
1. 从栈中加载输入颜色
2. 调用 `svm_separate_color` 将 RGB 颜色转换到目标颜色空间
3. 解包三个输出偏移量
4. 逐一检查有效性并将分量值存入栈

## 依赖关系

- **内部头文件**：`kernel/svm/color_util.h`、`kernel/svm/util.h`
- **被引用**：`kernel/svm/svm.h`（SVM 主调度器）

## 实现细节 / 关键算法

- **颜色空间转换**：实际的颜色空间转换逻辑封装在 `color_util.h` 中的 `svm_combine_color` 和 `svm_separate_color` 函数内，本文件仅负责 SVM 栈的数据流管理。
- **条件输出**：分离节点中每个输出分量都通过 `stack_valid` 检查，只有实际连接的输出才会被写入栈。这是一种常见的性能优化策略。
- **对称设计**：合并和分离节点是一对互补操作——合并节点将分量组合为颜色，分离节点将颜色拆分为分量。它们共享相同的颜色类型枚举 `NodeCombSepColorType`。

## 关联文件

- `kernel/svm/color_util.h`：提供 `svm_combine_color` 和 `svm_separate_color` 颜色空间转换函数
- `kernel/svm/svm.h`：SVM 主调度入口
- `kernel/svm/util.h`：栈操作与节点解包工具
