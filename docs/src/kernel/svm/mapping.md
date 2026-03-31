# mapping.h - 映射节点与纹理映射的着色器虚拟机实现

## 概述

本文件实现了着色器虚拟机（SVM）中的映射（Mapping）节点和纹理映射相关功能。映射节点用于对向量进行平移、旋转和缩放变换，是纹理坐标操控的核心工具。此外还提供了纹理映射矩阵变换和向量最小/最大值裁剪功能。

## 核心函数

### `svm_node_mapping`

```c
ccl_device_noinline void svm_node_mapping(
    ccl_private float *stack,
    const uint type,
    const uint inputs_stack_offsets,
    const uint result_stack_offset)
```

**功能**：对向量执行映射变换（点、纹理、向量或法线模式）。

**参数**：
- `stack`：SVM 栈指针
- `type`：映射类型，对应 `NodeMappingType` 枚举
- `inputs_stack_offsets`：打包的输入栈偏移量（向量、位置、旋转、缩放）
- `result_stack_offset`：结果输出的栈偏移量

**执行流程**：
1. 使用 `svm_unpack_node_uchar4` 解包四个输入偏移量
2. 从栈中加载向量、位置、旋转和缩放参数
3. 调用 `svm_mapping` 工具函数执行实际变换
4. 将结果存入栈

### `svm_node_texture_mapping`

```c
ccl_device_noinline int svm_node_texture_mapping(
    KernelGlobals kg,
    ccl_private float *stack,
    const uint vec_offset,
    const uint out_offset,
    int offset)
```

**功能**：使用 3x4 变换矩阵对向量执行纹理映射变换。

**执行流程**：
1. 从指令流中读取三行变换矩阵数据（`Transform`）
2. 使用 `transform_point` 对输入向量执行仿射变换

### `svm_node_min_max`

```c
ccl_device_noinline int svm_node_min_max(
    KernelGlobals kg,
    ccl_private float *stack,
    const uint vec_offset,
    const uint out_offset,
    int offset)
```

**功能**：将向量裁剪到指定的最小值和最大值范围内。

**执行流程**：
1. 从指令流中读取最小值和最大值
2. 对输入向量执行 `min(max(mn, v), mx)` 裁剪操作

## 依赖关系

- **内部头文件**：`kernel/svm/mapping_util.h`、`kernel/svm/util.h`
- **被引用**：`kernel/svm/svm.h`（SVM 主调度器）

## 实现细节 / 关键算法

- `svm_node_mapping` 将实际的数学变换委托给 `mapping_util.h` 中的 `svm_mapping` 函数，保持了节点接口层与算法层的分离。
- `svm_node_texture_mapping` 使用完整的 3x4 仿射变换矩阵（3 个 `float4` 行），从指令流中连续读取三个节点来构建矩阵。
- `svm_node_min_max` 中的最小值和最大值以 `float4` 节点的形式读取，但仅使用 xyz 三个分量来构造 `float3`。

## 关联文件

- `kernel/svm/mapping_util.h`：包含 `svm_mapping` 核心变换算法
- `kernel/svm/svm.h`：SVM 主调度入口
- `kernel/svm/util.h`：栈操作与节点解包工具
