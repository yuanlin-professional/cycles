# bump.h - 凹凸贴图评估进入/退出节点的着色器虚拟机实现

## 概述

本文件实现了着色器虚拟机（SVM）中凹凸贴图（Bump Mapping）评估的进入和退出节点。这对节点用于在评估凹凸贴图之前保存着色点的原始状态（位置和法线），并在凹凸贴图连接到置换时使用未置换的位置和法线，评估完毕后恢复原始状态。它是凹凸法线计算流程的关键基础设施。

## 核心函数

### `svm_node_enter_bump_eval`

```c
ccl_device_noinline void svm_node_enter_bump_eval(
    KernelGlobals kg,
    ccl_private ShaderData *sd,
    ccl_private float *stack,
    const uint offset)
```

**功能**：进入凹凸贴图评估状态，保存当前着色器数据并设置未置换的几何数据。

**参数**：
- `kg`：内核全局数据
- `sd`：着色器数据（ShaderData），包含当前着色点信息
- `stack`：SVM 栈指针
- `offset`：栈中保存状态的起始偏移量

**执行流程**：
1. 将当前位置 `sd->P` 和微分 `sd->dP` 保存到栈中（偏移量 0-3）
2. 查找 `ATTR_STD_POSITION_UNDISPLACED`（未置换位置）和 `ATTR_STD_NORMAL_UNDISPLACED`（未置换法线）属性
3. 如果找到未置换法线属性，将其归一化、变换到世界空间后设置为 `sd->N`
4. 如果找到未置换位置属性，将其变换到世界空间后设置为 `sd->P` 和 `sd->dP`
5. 将完整的位置微分（dx/dy）保存到栈中（偏移量 4-9），供后续 `svm_node_set_bump` 使用

### `svm_node_leave_bump_eval`

```c
ccl_device_noinline void svm_node_leave_bump_eval(
    ccl_private ShaderData *sd,
    ccl_private float *stack,
    const uint offset)
```

**功能**：退出凹凸贴图评估状态，恢复保存的着色器数据。

**参数**：
- `sd`：着色器数据
- `stack`：SVM 栈指针
- `offset`：栈中保存状态的起始偏移量

**执行流程**：
1. 从栈中恢复 `sd->P`（位置）和 `sd->dP`（微分）

## 依赖关系

- **内部头文件**：`kernel/globals.h`、`kernel/geom/attribute.h`、`kernel/geom/object.h`、`kernel/geom/primitive.h`、`kernel/svm/util.h`、`kernel/util/differential.h`
- **被引用**：`kernel/svm/svm.h`（SVM 主调度器）

## 实现细节 / 关键算法

- **栈布局**：在指定的 `offset` 开始，栈空间的布局为：
  - `[0-2]`：保存的 `sd->P`（float3，3 个槽位）
  - `[3]`：保存的 `sd->dP`（紧凑型微分，1 个槽位）
  - `[4-6]`：位置微分 `dP.dx`（float3）
  - `[7-9]`：位置微分 `dP.dy`（float3）

- **未置换几何的重要性**：当网格使用了置换贴图时，顶点位置已经被修改。凹凸贴图的评估需要基于原始（未置换）的几何数据来计算正确的法线扰动，否则会产生双重置换的错误效果。

- **循环查找属性的技巧**：代码使用一个长度为 2 的数组和 for 循环来查找两个属性，注释说明这是为了绕过 Metal 编译器的一个问题，使得只需要一组 `find_attribute` 和 `primitive_surface_attribute` 调用。

- **法线不需要恢复**：注释指出 `sd->N` 不需要在 `leave_bump_eval` 中恢复，因为后续的凹凸评估（`svm_node_set_bump`）会直接写入 `sd->N`。

- **完整微分 vs 紧凑型微分**：完整的微分（dx/dy 各为 float3）被保存在栈中，因为紧凑型微分对于 `svm_node_set_bump` 的计算不够精确（参见 bug #111588）。

## 关联文件

- `kernel/svm/displace.h`：包含 `svm_node_set_bump` 凹凸法线计算函数
- `kernel/svm/svm.h`：SVM 主调度入口
- `kernel/geom/attribute.h`：属性查找函数
- `kernel/geom/primitive.h`：图元表面属性插值函数
- `kernel/util/differential.h`：微分工具函数
