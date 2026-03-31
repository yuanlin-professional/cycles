# sepcomb_vector.h - 向量分离与合并节点的着色器虚拟机实现

## 概述

本文件实现了着色器虚拟机（SVM）中的向量合并（Combine Vector）和向量分离（Separate Vector）节点。这些节点用于在 float3 向量和单独的浮点分量之间进行转换，同时服务于 RGB 和 XYZ 节点。实现简洁高效，是着色器中最基础的数据类型转换工具。

## 核心函数

### `svm_node_combine_vector`

```c
ccl_device void svm_node_combine_vector(
    ccl_private float *stack,
    const uint in_offset,
    const uint vector_index,
    const uint out_offset)
```

**功能**：将单个浮点值写入向量的指定分量位置。

**参数**：
- `stack`：SVM 栈指针
- `in_offset`：输入浮点值的栈偏移量
- `vector_index`：目标分量索引（0=X, 1=Y, 2=Z）
- `out_offset`：输出向量的栈偏移量

**执行流程**：
1. 从栈中加载浮点值
2. 通过 `out_offset + vector_index` 直接定位到向量的指定分量位置并写入

### `svm_node_separate_vector`

```c
ccl_device void svm_node_separate_vector(
    ccl_private float *stack,
    const uint ivector_offset,
    const uint vector_index,
    const uint out_offset)
```

**功能**：从向量中提取指定分量。

**参数**：
- `stack`：SVM 栈指针
- `ivector_offset`：输入向量的栈偏移量
- `vector_index`：要提取的分量索引（0=X, 1=Y, 2=Z）
- `out_offset`：输出浮点值的栈偏移量

**执行流程**：
1. 从栈中加载 float3 向量
2. 根据 `vector_index` 的值（0/1/2）选择对应的分量（x/y/z）
3. 将该分量以浮点数形式存入栈

## 依赖关系

- **内部头文件**：`kernel/svm/util.h`
- **被引用**：`kernel/svm/svm.h`（SVM 主调度器）

## 实现细节 / 关键算法

- **合并节点的栈内存布局**：`svm_node_combine_vector` 利用了 SVM 栈中 float3 的内存连续性，通过 `out_offset + vector_index` 直接写入目标分量位置，而无需先加载整个向量。这意味着该节点需要被调用三次（分别写入 X、Y、Z）才能构成完整的向量。
- **分离节点的分支选择**：`svm_node_separate_vector` 通过 if-else 链选择分量，而非使用指针偏移。这在 GPU 上可能更友好，因为避免了运行时的间接内存访问。
- **轻量级函数**：两个函数都使用 `ccl_device`（而非 `ccl_device_noinline`），允许编译器内联，适合高频调用的简单操作。
- **双用途设计**：如代码注释所述，这些节点同时服务于 RGB（颜色）和 XYZ（向量）的分离/合并，因为两者在底层都是 float3。

## 关联文件

- `kernel/svm/svm.h`：SVM 主调度入口
- `kernel/svm/util.h`：栈操作工具
- `kernel/svm/sepcomb_color.h`：功能类似但带颜色空间转换的颜色分离/合并节点
