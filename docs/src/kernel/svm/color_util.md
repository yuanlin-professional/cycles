# color_util.h - 颜色混合与颜色空间转换工具函数库

## 概述

`color_util.h` 是 Cycles SVM 中颜色处理的核心工具库，提供了 19 种颜色混合模式的实现、亮度/对比度调整函数，以及 RGB/HSV/HSL 颜色空间的合并与分离功能。该文件被多个 SVM 节点文件引用，是着色器颜色运算的基础设施。

## 核心函数

### 颜色混合函数

所有混合函数签名形式为 `svm_mix_XXX(float t, float3 col1, float3 col2)`，其中 `t` 为混合因子，`col1` 为基色，`col2` 为混合色。

| 函数名 | 混合模式 | 算法简述 |
|--------|---------|---------|
| `svm_mix_blend` | 混合(Blend) | `interp(col1, col2, t)` 线性插值 |
| `svm_mix_add` | 相加(Add) | `interp(col1, col1+col2, t)` |
| `svm_mix_mul` | 相乘(Multiply) | `interp(col1, col1*col2, t)` |
| `svm_mix_screen` | 滤色(Screen) | `1 - (tm + t*(1-col2)) * (1-col1)` |
| `svm_mix_overlay` | 叠加(Overlay) | 分通道：<0.5 时暗化，>=0.5 时亮化 |
| `svm_mix_sub` | 相减(Subtract) | `interp(col1, col1-col2, t)` |
| `svm_mix_div` | 相除(Divide) | 分通道安全除法 |
| `svm_mix_diff` | 差值(Difference) | `interp(col1, abs(col1-col2), t)` |
| `svm_mix_exclusion` | 排除(Exclusion) | `col1 + col2 - 2*col1*col2`，结果钳位 >= 0 |
| `svm_mix_dark` | 变暗(Darken) | `interp(col1, min(col1, col2), t)` |
| `svm_mix_light` | 变亮(Lighten) | `interp(col1, max(col1, col2), t)` |
| `svm_mix_dodge` | 减淡(Dodge) | 分通道 `col1 / (1 - t*col2)` 并钳位 |
| `svm_mix_burn` | 加深(Burn) | 分通道 `1 - (1-col1) / (tm + t*col2)` 并钳位 |
| `svm_mix_hue` | 色相(Hue) | 在 HSV 空间替换色相分量 |
| `svm_mix_sat` | 饱和度(Saturation) | 在 HSV 空间插值饱和度分量 |
| `svm_mix_val` | 明度(Value) | 在 HSV 空间插值明度分量 |
| `svm_mix_color` | 颜色(Color) | 在 HSV 空间替换色相+饱和度分量 |
| `svm_mix_soft` | 柔光(Soft Light) | `(1-col1)*col2*col1 + col1*screen` |
| `svm_mix_linear` | 线性光(Linear Light) | `col1 + t*(2*col2 - 1)` |

### 混合总调度函数

```c
ccl_device_noinline_cpu float3 svm_mix(NodeMix type, float t, float3 c1, float3 c2)
```

- **功能**: 根据 `NodeMix` 枚举类型分派到具体的混合函数。

```c
ccl_device_noinline_cpu float3 svm_mix_clamped_factor(NodeMix type, float t, float3 c1, float3 c2)
```

- **功能**: 先将混合因子 `t` 钳位到 [0, 1]，再调用 `svm_mix()`。

### 其他工具函数

```c
ccl_device float3 svm_mix_clamp(const float3 col)
```

- **功能**: 将颜色各通道钳位到 [0, 1]（`saturate`）。

```c
ccl_device_inline float3 svm_brightness_contrast(float3 color, float brightness, float contrast)
```

- **功能**: 亮度/对比度调整。公式为 `max(a * color + b, 0)`，其中 `a = 1 + contrast`，`b = brightness - contrast * 0.5`。

```c
ccl_device float3 svm_combine_color(NodeCombSepColorType type, const float3 color)
ccl_device float3 svm_separate_color(NodeCombSepColorType type, const float3 color)
```

- **功能**: 颜色空间合并/分离。支持 RGB（直通）、HSV（`hsv_to_rgb` / `rgb_to_hsv`）和 HSL（`hsl_to_rgb` / `rgb_to_hsl`）三种模式。

## 依赖关系

- **内部头文件**:
  - `kernel/svm/types.h` — `NodeMix`、`NodeCombSepColorType` 枚举
  - `util/color.h` — `rgb_to_hsv`、`hsv_to_rgb`、`hsl_to_rgb`、`rgb_to_hsl` 等颜色空间转换函数
- **被引用**:
  - `kernel/svm/mix.h` — 混合节点入口
  - `kernel/svm/brightness.h` — 亮度/对比度节点
  - `kernel/svm/sepcomb_color.h` — 颜色合并/分离节点
  - `scene/shader_nodes.cpp` — 场景层着色器节点

## 实现细节 / 关键算法

### 混合因子语义

混合因子 `t` 统一含义为：`t=0` 时输出纯 `col1`，`t=1` 时输出完全混合后的效果。大多数模式使用 `interp(col1, result, t)` 的形式实现渐进混合。

### HSV 空间混合

色相(Hue)、饱和度(Saturation)、明度(Value)和颜色(Color)四种模式需要先将 RGB 转换到 HSV 空间进行操作，再转回 RGB。其中色相和颜色模式会检查源色饱和度是否为零（灰色），若为零则跳过混合以避免未定义的色相值。

### CPU/GPU 兼容

`svm_mix()` 和 `svm_mix_clamped_factor()` 使用 `ccl_device_noinline_cpu` 修饰符，表示在 CPU 上不内联（减少代码膨胀），在 GPU 上由编译器自行决定。

## 关联文件

- `kernel/svm/mix.h` — 混合节点 SVM 入口
- `kernel/svm/brightness.h` — 亮度对比度节点
- `kernel/svm/sepcomb_color.h` — 颜色合并/分离节点
- `util/color.h` — 底层颜色空间转换
