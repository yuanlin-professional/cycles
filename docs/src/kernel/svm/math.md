# math.h - 着色器虚拟机数学运算节点入口

## 概述

`math.h` 是 Cycles 着色器虚拟机（SVM）中标量数学节点和向量数学节点的主入口文件。它负责从 SVM 栈中加载操作数，调用 `math_util.h` 中的实际运算函数，并将结果写回栈中。该文件对应 Blender 中的"数学（Math）"节点和"向量数学（Vector Math）"节点。

## 核心函数

### `svm_node_math`

```c
ccl_device_noinline void svm_node_math(ccl_private float *stack,
                                       const uint type,
                                       const uint inputs_stack_offsets,
                                       const uint result_stack_offset)
```

- **功能**: 标量数学节点的 SVM 执行入口。
- **参数**:
  - `stack`: SVM 栈指针。
  - `type`: 数学运算类型，对应 `NodeMathType` 枚举（加、减、乘、除、幂、三角函数等）。
  - `inputs_stack_offsets`: 打包的三个输入参数栈偏移量（a, b, c），通过 `svm_unpack_node_uchar3` 解包。
  - `result_stack_offset`: 结果写入栈的偏移量。
- **流程**: 从栈加载三个浮点数 a、b、c，调用 `svm_math()` 执行运算，将标量结果存回栈。

### `svm_node_vector_math`

```c
ccl_device_noinline int svm_node_vector_math(KernelGlobals kg,
                                             ccl_private float *stack,
                                             const uint type,
                                             const uint inputs_stack_offsets,
                                             const uint outputs_stack_offsets,
                                             int offset)
```

- **功能**: 向量数学节点的 SVM 执行入口。
- **参数**:
  - `kg`: 内核全局数据，用于读取额外的 SVM 指令节点。
  - `type`: 向量运算类型，对应 `NodeVectorMathType` 枚举。
  - `inputs_stack_offsets`: 打包的输入栈偏移量（向量 a、b 和标量 param1）。
  - `outputs_stack_offsets`: 打包的输出栈偏移量（标量 value 和向量 vector）。
  - `offset`: 当前 SVM 指令偏移量。
- **流程**: 对于三操作数运算（`WRAP`、`FACEFORWARD`、`MULTIPLY_ADD`），需要读取额外的 SVM 节点来获取第三个向量 c。调用 `svm_vector_math()` 执行运算后，根据输出偏移量是否有效，分别存储标量和/或向量结果。
- **返回值**: 更新后的 SVM 指令偏移量。

## 依赖关系

- **内部头文件**:
  - `kernel/svm/math_util.h` — 提供 `svm_math()` 和 `svm_vector_math()` 实际运算实现
  - `kernel/svm/util.h` — 提供栈操作和节点解包工具函数
- **被引用**:
  - `kernel/svm/svm.h` — SVM 主调度器，在 `NODE_MATH` 和 `NODE_VECTOR_MATH` 指令分支中调用本文件的函数

## 实现细节

1. **栈参数打包**: 多个栈偏移量被打包到单个 `uint` 中，通过 `svm_unpack_node_uchar3` / `svm_unpack_node_uchar2` 解包，每个偏移量占一个字节（最多 256 个栈位置）。
2. **额外节点读取**: 向量数学中的三操作数运算（WRAP、FACEFORWARD、MULTIPLY_ADD）需要通过 `read_node()` 从 SVM 字节码流中读取额外的节点数据来获取第三个输入向量。
3. **双输出模式**: `svm_node_vector_math` 同时支持标量输出（如点积、距离、长度）和向量输出（如加法、叉积），通过 `stack_valid()` 检查是否需要存储对应输出。

## 关联文件

- `kernel/svm/math_util.h` — 标量和向量数学运算的核心实现
- `kernel/svm/svm.h` — SVM 指令调度器
- `kernel/svm/types.h` — `NodeMathType` 和 `NodeVectorMathType` 枚举定义
