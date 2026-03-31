# camera.h - 相机数据节点的着色器虚拟机实现

## 概述

本文件实现了着色器虚拟机（SVM）中的相机数据（Camera Data）节点。该节点提供着色点相对于相机的空间信息，包括观察方向向量、Z 深度和到相机的距离。这些数据常用于距离衰减效果、深度雾、视角相关的着色等场景。

## 核心函数

### `svm_node_camera`

```c
ccl_device_noinline void svm_node_camera(
    KernelGlobals kg,
    ccl_private ShaderData *sd,
    ccl_private float *stack,
    const uint out_vector,
    const uint out_zdepth,
    const uint out_distance)
```

**功能**：计算并输出着色点在相机空间中的观察信息。

**参数**：
- `kg`：内核全局数据，包含相机变换矩阵
- `sd`：着色器数据，包含当前着色点的世界空间位置
- `stack`：SVM 栈指针
- `out_vector`：观察方向向量的输出栈偏移量
- `out_zdepth`：Z 深度的输出栈偏移量
- `out_distance`：距离的输出栈偏移量

**执行流程**：
1. 从 `kernel_data.cam.worldtocamera` 获取世界到相机的变换矩阵
2. 使用 `transform_point` 将着色点位置 `sd->P` 变换到相机空间
3. 提取 Z 分量作为 Z 深度（沿相机光轴的距离）
4. 计算向量长度作为到相机的欧几里得距离
5. 按需输出：
   - **观察方向**（`out_vector`）：归一化的相机空间位置向量
   - **Z 深度**（`out_zdepth`）：相机空间中的 Z 坐标值
   - **距离**（`out_distance`）：着色点到相机的直线距离

## 依赖关系

- **内部头文件**：`kernel/globals.h`、`kernel/svm/util.h`
- **被引用**：`kernel/svm/svm.h`（SVM 主调度器）

## 实现细节 / 关键算法

- **Z 深度 vs 距离**：Z 深度是相机空间中沿光轴（Z 轴）的投影距离，距离则是三维空间中的真实欧几里得距离。对于位于相机光轴旁边的点，Z 深度小于实际距离。Z 深度对应的是渲染中常用的深度缓冲值。

- **条件输出**：三个输出都通过 `stack_valid` 检查，只有实际被下游节点使用的输出才会被计算和写入栈。这是标准的 SVM 优化模式。

- **观察方向的归一化**：输出的观察方向向量是归一化后的相机空间位置向量，指向从相机到着色点的方向，长度为 1。

- **简洁实现**：这是一个非常简洁的节点实现，只需一次矩阵变换即可获得所有三个输出值，计算效率较高。

## 关联文件

- `kernel/svm/svm.h`：SVM 主调度入口
- `kernel/svm/util.h`：栈操作工具
- `kernel/globals.h`：内核全局数据结构（包含 `kernel_data.cam`）
- `kernel/svm/tex_coord.h`：纹理坐标节点中也有相机空间变换（`svm_texco_camera`）
