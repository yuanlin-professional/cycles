# invert.h - 颜色反转节点

## 概述

`invert.h` 实现了 Cycles SVM 中的反转（Invert）节点，对应 Blender 着色器编辑器中的"反转（Invert）"节点。该节点将输入颜色的每个通道进行反转（取补），并通过混合因子控制反转程度，可用于创建底片效果或反转遮罩。

## 核心函数

### `invert`

```c
ccl_device float invert(const float color, const float factor)
```

- **功能**: 单通道颜色反转。
- **公式**: `result = factor * (1 - color) + (1 - factor) * color`
- **语义**: `factor=1` 时完全反转，`factor=0` 时保持原值，中间值为原色和反转色的线性混合。

### `svm_node_invert`

```c
ccl_device_noinline void svm_node_invert(ccl_private float *stack,
                                         const uint in_fac,
                                         const uint in_color,
                                         const uint out_color)
```

- **功能**: 反转节点的 SVM 执行入口。
- **参数**:
  - `stack`: SVM 栈指针。
  - `in_fac`: 混合因子的栈偏移量。
  - `in_color`: 输入颜色的栈偏移量。
  - `out_color`: 输出颜色的栈偏移量。
- **流程**: 从栈加载因子和颜色，对 R、G、B 三个通道分别调用 `invert()` 函数，将结果存回栈。

## 依赖关系

- **内部头文件**:
  - `kernel/svm/util.h` — 栈操作工具
- **被引用**:
  - `kernel/svm/svm.h` — SVM 主调度器，在 `NODE_INVERT` 指令分支中调用

## 实现细节 / 关键算法

### 反转公式化简

`invert` 函数的公式可以化简为：

```
result = factor * (1 - color) + (1 - factor) * color
       = factor - factor * color + color - factor * color
       = factor + color * (1 - 2 * factor)
```

当 `factor = 1` 时：`result = 1 - color`（完全反转）。
当 `factor = 0` 时：`result = color`（不变）。
当 `factor = 0.5` 时：`result = 0.5`（所有颜色映射到中灰）。

### 自包含实现

该文件是少数不依赖其他 SVM 颜色工具文件的节点之一，反转逻辑完全在文件内实现，仅依赖基础的栈操作工具 `util.h`。

## 关联文件

- `kernel/svm/svm.h` — SVM 指令调度器
- `kernel/svm/gamma.h` — 另一种颜色变换（伽马校正也可实现类似反转效果，如 gamma=-1）
