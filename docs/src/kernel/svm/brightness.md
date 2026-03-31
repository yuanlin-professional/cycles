# brightness.h - 亮度/对比度调整节点

## 概述

`brightness.h` 实现了 Cycles SVM 中的亮度/对比度（Bright/Contrast）节点，对应 Blender 着色器编辑器中的"亮度/对比度（Bright/Contrast）"节点。该节点通过线性变换同时调整颜色的亮度和对比度，结果钳位为非负值。

## 核心函数

### `svm_node_brightness`

```c
ccl_device_noinline void svm_node_brightness(ccl_private float *stack,
                                             const uint in_color,
                                             const uint out_color,
                                             const uint node)
```

- **功能**: 亮度/对比度节点的 SVM 执行入口。
- **参数**:
  - `stack`: SVM 栈指针。
  - `in_color`: 输入颜色的栈偏移量。
  - `out_color`: 输出颜色的栈偏移量。
  - `node`: 打包的额外参数，通过 `svm_unpack_node_uchar2` 解包为亮度偏移 `bright_offset` 和对比度偏移 `contrast_offset`。
- **流程**: 从栈加载输入颜色、亮度值和对比度值，调用 `svm_brightness_contrast()` 进行调整，将结果存回栈。

## 依赖关系

- **内部头文件**:
  - `kernel/svm/color_util.h` — 提供 `svm_brightness_contrast()` 实际调整算法
  - `kernel/svm/util.h` — 栈操作和节点解包工具
- **被引用**:
  - `kernel/svm/svm.h` — SVM 主调度器，在 `NODE_BRIGHTCONTRAST` 指令分支中调用

## 实现细节 / 关键算法

### 亮度/对比度调整公式（`svm_brightness_contrast` 在 color_util.h 中）

```
a = 1 + contrast
b = brightness - contrast * 0.5
output = max(a * color + b, 0)
```

- **对比度(contrast)**: 值为 0 时不改变对比度。正值增大对比度（亮的更亮，暗的更暗），负值减小对比度。对比度通过乘以系数 `a = 1 + contrast` 来缩放颜色范围。
- **亮度(brightness)**: 值为 0 时不改变亮度。正值增加亮度，负值降低亮度。偏移量 `b = brightness - contrast * 0.5` 确保对比度调整以中间灰度（0.5）为中心。
- **结果钳位**: 逐通道取 `max(result, 0)`，防止结果出现负值，但不限制上限（允许超过 1.0 的 HDR 值）。

## 关联文件

- `kernel/svm/color_util.h` — `svm_brightness_contrast()` 的定义位置
- `kernel/svm/svm.h` — SVM 指令调度器
- `kernel/svm/gamma.h` — 另一种颜色调整方式（伽马校正）
