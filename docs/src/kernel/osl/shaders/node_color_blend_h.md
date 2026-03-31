# node_color_blend.h - 颜色混合模式辅助函数头文件

## 概述

该头文件为 Cycles 渲染器的开放着色语言（OSL）着色器提供各种颜色混合（Blend）模式的实现函数。支持 18 种混合模式，涵盖基本混合、光影效果、HSV 分量混合等，对应常见图像编辑软件中的图层混合模式。所有函数接受混合因子 `t`（范围 [0, 1]）以及两个输入颜色 `col1`（底色）和 `col2`（混合色）。依赖 `node_color.h` 中的 `rgb_to_hsv()` 和 `hsv_to_rgb()` 函数。

## 函数列表

### 基本混合模式

| 函数签名 | 混合模式 | 说明 |
|----------|----------|------|
| `color node_mix_blend(float t, color col1, color col2)` | 正常混合 | `mix(col1, col2, t)` -- 基本线性插值 |
| `color node_mix_add(float t, color col1, color col2)` | 相加 | `mix(col1, col1 + col2, t)` -- 颜色叠加增亮 |
| `color node_mix_mul(float t, color col1, color col2)` | 相乘 | `mix(col1, col1 * col2, t)` -- 颜色相乘变暗 |
| `color node_mix_sub(float t, color col1, color col2)` | 相减 | `mix(col1, col1 - col2, t)` -- 颜色差值变暗 |
| `color node_mix_div(float t, color col1, color col2)` | 相除 | 逐通道除法（避免除零） |
| `color node_mix_diff(float t, color col1, color col2)` | 差值 | `mix(col1, abs(col1 - col2), t)` -- 取绝对差 |
| `color node_mix_exclusion(float t, color col1, color col2)` | 排除 | 类似差值但对比度更低的混合 |

### 光影效果混合模式

| 函数签名 | 混合模式 | 说明 |
|----------|----------|------|
| `color node_mix_screen(float t, color col1, color col2)` | 滤色 | 反转相乘再反转，整体增亮 |
| `color node_mix_overlay(float t, color col1, color col2)` | 叠加 | 暗区相乘、亮区滤色的组合 |
| `color node_mix_dodge(float t, color col1, color col2)` | 减淡 | 增亮效果，可能产生高光溢出 |
| `color node_mix_burn(float t, color col1, color col2)` | 加深 | 加深暗部效果 |
| `color node_mix_soft(float t, color col1, color col2)` | 柔光 | 柔和的光照效果 |
| `color node_mix_linear(float t, color col1, color col2)` | 线性光 | 基于混合色的亮度线性调整 |

### 明暗选择混合模式

| 函数签名 | 混合模式 | 说明 |
|----------|----------|------|
| `color node_mix_dark(float t, color col1, color col2)` | 变暗 | `mix(col1, min(col1, col2), t)` -- 逐通道取较暗值 |
| `color node_mix_light(float t, color col1, color col2)` | 变亮 | `mix(col1, max(col1, col2), t)` -- 逐通道取较亮值 |

### HSV 分量混合模式

| 函数签名 | 混合模式 | 说明 |
|----------|----------|------|
| `color node_mix_hue(float t, color col1, color col2)` | 色相 | 只替换底色的色相分量 |
| `color node_mix_sat(float t, color col1, color col2)` | 饱和度 | 只替换底色的饱和度分量 |
| `color node_mix_val(float t, color col1, color col2)` | 明度 | 只替换底色的明度分量 |
| `color node_mix_color(float t, color col1, color col2)` | 颜色 | 替换底色的色相和饱和度，保留明度 |

### 辅助函数

| 函数签名 | 说明 |
|----------|------|
| `color node_mix_clamp(color col)` | 将颜色各通道钳制到 [0, 1] 范围 |

## 关键实现细节

### 叠加模式（Overlay）
对每个通道独立判断：
- 底色通道 < 0.5 时：使用相乘公式 `outcol *= tm + 2t * col2`。
- 底色通道 >= 0.5 时：使用滤色公式 `outcol = 1 - (tm + 2t * (1 - col2)) * (1 - outcol)`。

其中 `tm = 1 - t`。

### 减淡模式（Dodge）
对每个通道：
- 底色为零时保持为零（避免无意义的减淡）。
- 计算 `tmp = 1 - t * col2`，若 `tmp <= 0` 则输出 1.0（完全高光）。
- 否则输出 `min(outcol / tmp, 1.0)`。

### 加深模式（Burn）
对每个通道：
- 计算 `tmp = tm + t * col2`。
- 若 `tmp <= 0` 输出 0.0（完全黑色）。
- 否则计算 `1 - (1 - outcol) / tmp` 并钳制到 [0, 1]。

### HSV 分量混合
色相、饱和度、颜色混合模式仅在混合色饱和度不为零时执行替换，以避免对无色度颜色（灰色）产生不当影响。明度混合无此限制。

## 对应用途

这些函数被 Cycles 中的混合（Mix）着色器和颜色混合相关节点调用，对应 Blender 节点编辑器中"混合"（Mix）节点的各种混合模式选项。
