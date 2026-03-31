# vector_rotate.h - 向量旋转节点的着色器虚拟机实现

## 概述

本文件实现了着色器虚拟机（SVM）中的向量旋转（Vector Rotate）节点。该节点支持多种旋转模式，包括绕固定轴（X/Y/Z）旋转、绕自定义轴旋转以及基于欧拉角的自由旋转。所有旋转都可以指定旋转中心，并支持反转操作。

## 核心函数

### `svm_node_vector_rotate`

```c
ccl_device_noinline void svm_node_vector_rotate(
    ccl_private float *stack,
    const uint input_stack_offsets,
    const uint axis_stack_offsets,
    const uint result_stack_offset)
```

**功能**：对输入向量执行指定类型的旋转变换。

**参数**：
- `stack`：SVM 栈指针
- `input_stack_offsets`：打包的输入参数偏移量（类型、向量、旋转、反转标志）
- `axis_stack_offsets`：打包的轴参数偏移量（中心点、轴向量、角度）
- `result_stack_offset`：结果输出的栈偏移量

**支持的旋转类型**：

| 类型 | 枚举值 | 说明 |
|------|--------|------|
| 绕 X 轴 | `NODE_VECTOR_ROTATE_TYPE_AXIS_X` | 绕世界 X 轴旋转 |
| 绕 Y 轴 | `NODE_VECTOR_ROTATE_TYPE_AXIS_Y` | 绕世界 Y 轴旋转 |
| 绕 Z 轴 | `NODE_VECTOR_ROTATE_TYPE_AXIS_Z` | 绕世界 Z 轴旋转 |
| 自定义轴 | （default 分支） | 绕用户指定的任意轴旋转 |
| 欧拉 XYZ | `NODE_VECTOR_ROTATE_TYPE_EULER_XYZ` | 使用欧拉角进行完整的三轴旋转 |

**执行流程**：
1. 解包两组参数偏移量
2. 加载输入向量和旋转中心
3. 根据旋转类型选择不同的计算路径：
   - **欧拉模式**：构建旋转矩阵，以中心点为原点进行变换
   - **轴旋转模式**：确定旋转轴和角度，调用 `rotate_around_axis` 执行罗德里格斯旋转
4. 支持通过 `invert` 标志反转旋转方向

## 依赖关系

- **内部头文件**：`kernel/svm/util.h`
- **被引用**：`kernel/svm/svm.h`（SVM 主调度器）

## 实现细节 / 关键算法

- **围绕中心点旋转**：所有旋转模式都先将向量平移到以中心点为原点的坐标系（`vector - center`），旋转后再平移回去（`+ center`）。这允许用户指定旋转的枢轴点。
- **欧拉旋转的正反方向**：正向旋转使用 `transform_direction`，反向（`invert`）使用 `transform_direction_transposed`（转置矩阵等效于正交矩阵的逆）。
- **轴旋转的安全检查**：当自定义轴长度为零时（`axis_length == 0`），返回原始向量而不进行旋转，防止除零错误。
- **角度反转**：在轴旋转模式中，通过取反角度值实现反转（`invert ? -angle : angle`）。
- **罗德里格斯旋转公式**：`rotate_around_axis` 实现了绕任意轴旋转的标准算法。

## 关联文件

- `kernel/svm/svm.h`：SVM 主调度入口
- `kernel/svm/util.h`：栈操作与节点解包工具
- `util/transform.h`：提供 `euler_to_transform` 和 `rotate_around_axis` 函数
