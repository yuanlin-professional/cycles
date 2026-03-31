# vector_transform.h - 向量变换节点的着色器虚拟机实现

## 概述

本文件实现了着色器虚拟机（SVM）中的向量变换（Vector Transform）节点。该节点用于在世界空间、相机空间和物体空间之间转换向量，支持点（Point）、方向（Vector）和法线（Normal）三种变换类型。不同的变换类型使用不同的矩阵运算以确保几何正确性。

## 核心函数

### `svm_node_vector_transform`

```c
ccl_device_noinline void svm_node_vector_transform(
    KernelGlobals kg,
    ccl_private ShaderData *sd,
    ccl_private float *stack,
    const uint4 node)
```

**功能**：在三种坐标空间之间转换向量。

**参数**：
- `kg`：内核全局数据，包含相机变换矩阵等
- `sd`：着色器数据，包含物体变换信息
- `stack`：SVM 栈指针
- `node`：编码了变换类型、源空间、目标空间和栈偏移量的节点数据

**支持的变换类型**（`NodeVectorTransformType`）：

| 类型 | 说明 | 变换方式 |
|------|------|----------|
| Point | 位置点 | `transform_point` |
| Vector | 方向向量 | `transform_direction` |
| Normal | 法线 | `transform_direction_transposed`（逆转置） |

**支持的坐标空间**（`NodeVectorTransformConvertSpace`）：

| 空间 | 说明 |
|------|------|
| World | 世界空间 |
| Object | 物体局部空间 |
| Camera | 相机空间 |

**执行流程**：
1. 解包变换类型（`itype`）、源空间（`ifrom`）、目标空间（`ito`）和栈偏移量
2. 按源空间分三大分支处理：
   - **从世界空间**：转到相机空间使用相机矩阵，转到物体空间使用物体逆变换
   - **从相机空间**：先转到世界空间，再按需转到物体空间
   - **从物体空间**：先转到世界空间，再按需转到相机空间
3. 将结果存入栈

## 依赖关系

- **内部头文件**：`kernel/geom/object.h`、`kernel/svm/util.h`
- **被引用**：`kernel/svm/svm.h`（SVM 主调度器）

## 实现细节 / 关键算法

- **法线变换的特殊性**：法线不能直接使用同一变换矩阵。在非均匀缩放下，法线需要使用原变换矩阵的逆转置来变换。代码中通过 `transform_direction_transposed` 使用对应的转置矩阵（对正交矩阵等价于逆矩阵），并在变换后重新归一化。
- **两步转换策略**：相机空间到物体空间（或反向）的转换被分解为两步：先经过世界空间中转。例如，相机到物体 = 相机到世界 + 世界到物体。
- **物体存在性检查**：涉及物体空间的变换使用 `is_object`（即 `sd->object != OBJECT_NONE`）守护，确保在背景着色器等无物体上下文中不会执行无效的物体变换。
- **相机矩阵获取**：从 `kernel_data.cam.worldtocamera` 和 `kernel_data.cam.cameratoworld` 获取相机变换矩阵。

## 关联文件

- `kernel/svm/svm.h`：SVM 主调度入口
- `kernel/geom/object.h`：提供物体空间变换函数（`object_normal_transform`、`object_dir_transform` 等）
- `kernel/svm/util.h`：栈操作与节点解包工具
