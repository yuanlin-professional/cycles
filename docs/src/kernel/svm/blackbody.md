# blackbody.h - 黑体辐射颜色节点

## 概述

`blackbody.h` 实现了 Cycles SVM 中的黑体辐射（Blackbody）节点，对应 Blender 着色器编辑器中的"黑体（Blackbody）"节点。该节点根据输入的色温（开尔文）计算物理上对应的黑体辐射颜色，常用于模拟白炽灯、火焰、恒星等热辐射光源的颜色。代码改编自 Open Shading Language（OSL）。

## 核心函数

### `svm_node_blackbody`

```c
ccl_device_noinline void svm_node_blackbody(KernelGlobals kg,
                                            ccl_private float *stack,
                                            const uint temperature_offset,
                                            const uint col_offset)
```

- **功能**: 黑体辐射节点的 SVM 执行入口。
- **参数**:
  - `kg`: 内核全局数据，用于颜色空间转换。
  - `stack`: SVM 栈指针。
  - `temperature_offset`: 色温输入值的栈偏移量（单位：开尔文）。
  - `col_offset`: 输出颜色的栈偏移量。
- **流程**:
  1. 从栈加载色温值。
  2. 调用 `svm_math_blackbody_color_rec709()` 计算 Rec.709 色彩空间下的黑体颜色。
  3. 通过 `rec709_to_rgb()` 转换到当前渲染使用的 RGB 色彩空间。
  4. 钳位为非负值（`max(color, 0)`）。
  5. 将结果写入栈。

## 依赖关系

- **内部头文件**:
  - `kernel/globals.h` — 内核全局数据结构定义
  - `kernel/svm/math_util.h` — 提供 `svm_math_blackbody_color_rec709()` 黑体颜色近似计算
  - `kernel/svm/util.h` — 栈操作工具
  - `kernel/util/colorspace.h` — 提供 `rec709_to_rgb()` 颜色空间转换
- **被引用**:
  - `kernel/svm/svm.h` — SVM 主调度器，在 `NODE_BLACKBODY` 指令分支中调用

## 实现细节 / 关键算法

### 黑体辐射物理背景

黑体辐射是理想黑体在热平衡状态下发出的电磁辐射。其光谱分布由普朗克定律决定，辐射颜色随温度变化：低温（~1000K）呈暗红色，中温（~3000K）呈橙黄色，高温（~6500K）呈近白色，极高温（>10000K）呈蓝白色。

### 计算流程

1. **Rec.709 近似**（在 `math_util.h` 中）: 使用 7 段分段有理函数近似，有效范围 800K-12000K。R/G 通道使用 `a/T + b*T + c` 形式，B 通道使用三次多项式。
2. **色彩空间转换**: 近似结果为 Rec.709/sRGB 色彩空间，需通过 `rec709_to_rgb()` 转换到场景的工作色彩空间（如 ACEScg 等）。
3. **负值钳位**: 由于近似公式可能产生负值（尤其在极端色温下），最终结果需钳位到非负。

### 许可证

本文件使用 BSD-3-Clause 许可证（与大多数 SVM 文件的 Apache-2.0 不同），因为代码改编自 Sony Pictures Imageworks 的 Open Shading Language。

## 关联文件

- `kernel/svm/math_util.h` — `svm_math_blackbody_color_rec709()` 的定义位置
- `kernel/svm/wavelength.h` — 另一种物理颜色转换（波长到 RGB）
- `kernel/util/colorspace.h` — 颜色空间转换矩阵
- `kernel/tables.h` — 黑体辐射查找表 `blackbody_table_r/g/b`
