# value.h - 常量值节点的着色器虚拟机实现

## 概述

本文件实现了着色器虚拟机（SVM）中的常量值（Value）节点，包括浮点数值节点和向量值节点。这些节点将编译时确定的常量值加载到 SVM 栈中，是着色器节点图中所有字面量输入的底层实现。它们是最基础的 SVM 节点之一。

## 核心函数

### `svm_node_value_f`

```c
ccl_device void svm_node_value_f(
    ccl_private float *stack,
    const uint ivalue,
    const uint out_offset)
```

**功能**：将一个浮点常量值压入 SVM 栈。

**参数**：
- `stack`：SVM 栈指针
- `ivalue`：以 uint 编码的浮点值
- `out_offset`：输出的栈偏移量

**执行流程**：
1. 使用 `__uint_as_float` 将 uint 位模式重新解释为 float
2. 将浮点值存入栈的指定位置

### `svm_node_value_v`

```c
ccl_device int svm_node_value_v(
    KernelGlobals kg,
    ccl_private float *stack,
    const uint out_offset,
    int offset)
```

**功能**：将一个 float3 向量常量值压入 SVM 栈。

**参数**：
- `kg`：内核全局数据，用于读取额外节点数据
- `stack`：SVM 栈指针
- `out_offset`：输出的栈偏移量
- `offset`：当前指令流偏移量

**执行流程**：
1. 通过 `read_node` 从指令流读取一个额外的 `uint4` 节点
2. 将 `node1.y`、`node1.z`、`node1.w` 三个分量从 uint 转换为 float
3. 构造 float3 向量并存入栈

**返回值**：更新后的指令流偏移量。

## 依赖关系

- **内部头文件**：`kernel/svm/util.h`
- **被引用**：`kernel/svm/svm.h`（SVM 主调度器）

## 实现细节 / 关键算法

- **位模式重解释**：浮点值以 uint 形式嵌入 SVM 指令流中，通过 `__uint_as_float` 进行无损的位模式重解释（而非类型转换）。这保证了常量值的精度不会丢失。

- **浮点 vs 向量的编码差异**：
  - 浮点常量直接编码在节点的参数字段中，无需额外读取
  - 向量常量需要一个额外的 `uint4` 节点来存储三个分量（使用 y/z/w 三个字段，x 字段未使用）

- **轻量级实现**：两个函数都使用 `ccl_device`（允许内联），因为操作极其简单。浮点值节点甚至不需要 `KernelGlobals` 参数，因为值直接编码在指令参数中。

- **在节点图编译中的角色**：当用户在着色器编辑器中设置数值输入时，编译器会生成这些常量值节点。如果输入通过连线接收数据，则不会生成常量值节点，而是由上游节点直接写入栈。

## 关联文件

- `kernel/svm/svm.h`：SVM 主调度入口
- `kernel/svm/util.h`：栈操作工具（`stack_store_float`、`stack_store_float3`）
