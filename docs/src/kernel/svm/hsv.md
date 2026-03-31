# hsv.h - HSV 色相/饱和度/明度调整节点

## 概述

`hsv.h` 实现了 Cycles SVM 中的 HSV（色相-饱和度-明度）节点，对应 Blender 着色器编辑器中的"色相饱和度明度（Hue Saturation Value）"节点。该节点允许用户在 HSV 色彩空间中对输入颜色的色相、饱和度和明度进行独立调整，并支持通过混合因子控制调整强度。

## 核心函数

### `svm_node_hsv`

```c
ccl_device_noinline void svm_node_hsv(ccl_private float *stack, const uint4 node)
```

- **功能**: HSV 颜色调整节点的 SVM 执行入口。
- **参数**:
  - `stack`: SVM 栈指针。
  - `node`: 打包的 SVM 节点数据，包含所有输入/输出栈偏移量。
    - `node.y` 解包为：输入颜色偏移、混合因子偏移、输出颜色偏移
    - `node.z` 解包为：色相偏移、饱和度偏移、明度偏移
- **输入**:
  - `fac`: 混合因子，控制效果强度
  - `in_color`: 输入 RGB 颜色
  - `hue`: 色相偏移量（加性，0.5 为中性值）
  - `sat`: 饱和度缩放因子（乘性，1.0 为不变）
  - `val`: 明度缩放因子（乘性，1.0 为不变）
- **输出**: 调整后的 RGB 颜色

## 依赖关系

- **内部头文件**:
  - `kernel/svm/util.h` — 栈操作和节点解包工具
  - `util/color.h` — `rgb_to_hsv()` 和 `hsv_to_rgb()` 颜色空间转换
- **被引用**:
  - `kernel/svm/svm.h` — SVM 主调度器，在 `NODE_HSV` 指令分支中调用

## 实现细节 / 关键算法

### 处理流程

1. **RGB 转 HSV**: 将输入颜色从 RGB 转换到 HSV 空间。
2. **色相偏移**: `H = fract(H + hue + 0.5)`。加 0.5 使得默认值 0.5 表示无偏移，使用 `fractf()` 确保结果循环在 [0, 1) 范围内。
3. **饱和度缩放**: `S = saturate(S * sat)`。乘以饱和度因子后钳位到 [0, 1]，防止过饱和。
4. **明度缩放**: `V = V * val`。简单乘法，不钳位（允许超亮）。
5. **HSV 转回 RGB**: 将调整后的 HSV 转换回 RGB。
6. **因子混合**: `output = fac * adjusted + (1 - fac) * original`，逐通道插值。
7. **负值钳位**: 最终对每个通道取 `max(0)`，防止过饱和导致的负值。

### 色相偏移的环绕处理

色相是循环量（0 和 1 表示同一颜色），使用 `fractf()` 取小数部分实现自然的环绕效果。默认偏移量 0.5 对应"不偏移"（因为 `fract(H + 0.5 + 0.5) = fract(H + 1) = H`）。

## 关联文件

- `kernel/svm/svm.h` — SVM 指令调度器
- `util/color.h` — HSV/RGB 颜色空间转换
- `kernel/svm/color_util.h` — 其他颜色混合模式中也使用 HSV 转换
