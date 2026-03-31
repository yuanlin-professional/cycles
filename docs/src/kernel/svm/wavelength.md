# wavelength.h - 波长转 RGB 颜色节点

## 概述

`wavelength.h` 实现了 Cycles SVM 中的波长转 RGB（Wavelength）节点，对应 Blender 着色器编辑器中的"波长（Wavelength）"节点。该节点将可见光波长（纳米）转换为对应的 RGB 颜色，可用于模拟棱镜色散、彩虹等光学现象。代码改编自 Open Shading Language（OSL）。

## 核心函数

### `svm_node_wavelength`

```c
ccl_device_noinline void svm_node_wavelength(KernelGlobals kg,
                                             ccl_private float *stack,
                                             const uint wavelength,
                                             const uint color_out)
```

- **功能**: 波长转 RGB 节点的 SVM 执行入口。
- **参数**:
  - `kg`: 内核全局数据，用于颜色空间转换。
  - `stack`: SVM 栈指针。
  - `wavelength`: 波长输入值的栈偏移量（单位：纳米）。
  - `color_out`: 输出颜色的栈偏移量。
- **流程**:
  1. 从栈加载波长值。
  2. 调用 `svm_math_wavelength_color_xyz()` 将波长转换为 CIE XYZ 颜色。
  3. 通过 `xyz_to_rgb()` 从 XYZ 转换到当前渲染使用的 RGB 色彩空间。
  4. 乘以经验缩放因子 `1/2.52`，使所有分量不超过 1.0。
  5. 钳位为非负值。
  6. 将结果写入栈。

## 依赖关系

- **内部头文件**:
  - `kernel/svm/math_util.h` — 提供 `svm_math_wavelength_color_xyz()` CIE XYZ 查表函数
  - `kernel/svm/util.h` — 栈操作工具
  - `kernel/util/colorspace.h` — 提供 `xyz_to_rgb()` 颜色空间转换
- **被引用**:
  - `kernel/svm/svm.h` — SVM 主调度器，在 `NODE_WAVELENGTH` 指令分支中调用

## 实现细节 / 关键算法

### CIE 1931 色彩匹配函数

`svm_math_wavelength_color_xyz()`（在 `math_util.h` 中）使用 CIE 1931 标准观察者色彩匹配函数的查找表。表覆盖 380nm 至 780nm 的可见光范围，以 5nm 为步长存储 80 个采样点（每点含 X、Y、Z 三个分量）。查找时使用线性插值获取任意波长的精确值。

### 经验缩放因子

缩放因子 `1.0 / 2.52` 是经验值，目的是将 CIE XYZ 到 RGB 转换后的最大分量值归一化到接近 1.0。注释中提到这是"Empirical scale from lg to make all comps <= 1"。

### 色彩空间转换链

```
波长(nm) -> CIE XYZ -> RGB(工作空间)
```

从波长到最终 RGB 需要经过两步转换：先通过查表得到 CIE XYZ，再通过场景配置的颜色转换矩阵转到工作 RGB 空间。

### 超出范围处理

波长超出可见光范围（<380nm 或 >780nm）时，`svm_math_wavelength_color_xyz()` 返回黑色 (0,0,0)，对应人眼无法感知的紫外线和红外线。

### 许可证

与 `blackbody.h` 相同，使用 BSD-3-Clause 许可证，源自 Open Shading Language。

## 关联文件

- `kernel/svm/math_util.h` — `svm_math_wavelength_color_xyz()` 的定义位置和 CIE 查找表
- `kernel/svm/blackbody.h` — 另一种物理颜色转换（色温到 RGB）
- `kernel/util/colorspace.h` — 颜色空间转换矩阵
- `kernel/tables.h` — CIE 色彩匹配查找表 `cie_color_match`
