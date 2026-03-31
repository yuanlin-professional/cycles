# normal.h - 法线节点的着色器虚拟机实现

## 概述

本文件实现了着色器虚拟机（SVM）中的法线（Normal）节点。该节点接收一个输入法线向量，输出一个预设的固定方向向量以及该方向与输入法线之间的点积值。这是 Blender 着色器节点编辑器中"法线"节点在 Cycles 内核中的底层实现。

## 核心函数

### `svm_node_normal`

```c
ccl_device_noinline int svm_node_normal(
    KernelGlobals kg,
    ccl_private float *stack,
    const uint in_normal_offset,
    const uint out_normal_offset,
    const uint out_dot_offset,
    int offset)
```

**功能**：执行法线节点的计算逻辑。

**参数**：
- `kg`：内核全局数据，用于读取额外的节点数据
- `stack`：SVM 栈指针
- `in_normal_offset`：输入法线在栈中的偏移量
- `out_normal_offset`：输出法线方向在栈中的偏移量
- `out_dot_offset`：输出点积值在栈中的偏移量
- `offset`：当前指令流偏移量

**执行流程**：
1. 通过 `read_node` 从指令流中读取预设方向向量（以 `uint4` 编码的 float3）
2. 对预设方向进行归一化
3. 如果 `out_normal_offset` 有效，将归一化的方向向量存入栈
4. 如果 `out_dot_offset` 有效，计算方向向量与归一化输入法线的点积并存入栈

**返回值**：更新后的指令流偏移量。

## 依赖关系

- **内部头文件**：`kernel/svm/util.h`
- **被引用**：`kernel/svm/svm.h`（SVM 主调度器）

## 实现细节 / 关键算法

- 预设方向向量以整数形式编码在额外的节点数据（`uint4`）中，通过 `__int_as_float` 逐分量转换为浮点数。这是 SVM 指令流中常见的数据打包方式。
- 输入法线和预设方向都会被归一化后再计算点积，确保点积结果在 [-1, 1] 范围内。
- 使用 `stack_valid` 检查输出偏移量的有效性，避免对未连接的输出进行不必要的写入操作。

## 关联文件

- `kernel/svm/svm.h`：SVM 主调度入口，调用本节点函数
- `kernel/svm/util.h`：提供栈操作工具函数（`stack_load_float3`、`stack_store_float3` 等）
- `kernel/svm/types.h`：定义 SVM 节点类型枚举
