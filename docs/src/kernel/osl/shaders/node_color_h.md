# node_color.h - 颜色空间转换辅助函数头文件

## 概述

该头文件为 Cycles 渲染器的开放着色语言（OSL）着色器提供颜色空间转换的核心函数。包含 sRGB 与线性空间的相互转换、颜色预乘还原、CIE xyY/XYZ 色彩空间转换，以及 RGB、HSV、HSL 三种常用色彩模型之间的转换函数。被 `node_hsv.osl`、`node_color_blend.h` 等多个着色器和头文件引用。

## 函数列表

### sRGB / 线性空间转换

| 函数签名 | 说明 |
|----------|------|
| `float color_srgb_to_scene_linear(float c)` | 单通道 sRGB 到线性空间转换 |
| `float color_scene_linear_to_srgb(float c)` | 单通道线性空间到 sRGB 转换 |
| `color color_srgb_to_scene_linear(color c)` | 三通道 sRGB 到线性空间转换 |
| `color color_scene_linear_to_srgb(color c)` | 三通道线性空间到 sRGB 转换 |

#### sRGB 到线性空间算法
```
if c < 0.04045:
    result = c / 12.92        (低值区线性段)
else:
    result = ((c + 0.055) / 1.055) ^ 2.4  (高值区伽马段)
```
负值输入返回 0.0。

#### 线性空间到 sRGB 算法
```
if c < 0.0031308:
    result = c * 12.92        (低值区线性段)
else:
    result = 1.055 * c ^ (1/2.4) - 0.055  (高值区伽马段)
```
负值输入返回 0.0。

### 颜色辅助函数

| 函数签名 | 说明 |
|----------|------|
| `color color_unpremultiply(color c, float alpha)` | 颜色预乘还原。当 alpha 不为 0 或 1 时，返回 `c / alpha` |

### CIE 色彩空间转换

| 函数签名 | 说明 |
|----------|------|
| `color xyY_to_xyz(float x, float y, float Y)` | CIE xyY 到 XYZ 色彩空间转换 |
| `color xyz_to_rgb(float x, float y, float z)` | CIE XYZ 到 RGB 色彩空间转换（使用 sRGB/BT.709 矩阵） |

#### XYZ 到 RGB 转换矩阵
使用 sRGB 标准的 3x3 转换矩阵：
```
R =  3.240479 * X - 1.537150 * Y - 0.498535 * Z
G = -0.969256 * X + 1.875991 * Y + 0.041556 * Z
B =  0.055648 * X - 0.204043 * Y + 1.057311 * Z
```

### RGB / HSV 转换

| 函数签名 | 说明 |
|----------|------|
| `color rgb_to_hsv(color rgb)` | RGB 到 HSV 色彩空间转换 |
| `color hsv_to_rgb(color hsv)` | HSV 到 RGB 色彩空间转换 |

#### rgb_to_hsv 算法
1. 计算 RGB 各通道的最大值 `cmax` 和最小值 `cmin`。
2. **明度（V）** = `cmax`。
3. **饱和度（S）** = `(cmax - cmin) / cmax`（`cmax == 0` 时 S = 0）。
4. **色相（H）**：根据哪个通道为最大值，在六个扇区中计算色相角并归一化到 [0, 1]。

#### hsv_to_rgb 算法
1. `S == 0` 时返回灰色 `(V, V, V)`。
2. 将色相 H 乘以 6 划分为六个扇区。
3. 计算插值参数 `p`、`q`、`t`。
4. 根据扇区索引选择 RGB 组合。

### RGB / HSL 转换

| 函数签名 | 说明 |
|----------|------|
| `color rgb_to_hsl(color rgb)` | RGB 到 HSL 色彩空间转换 |
| `color hsl_to_rgb(color hsl)` | HSL 到 RGB 色彩空间转换 |

#### rgb_to_hsl 算法
1. **亮度（L）** = `min(1.0, (cmax + cmin) / 2)`。
2. **饱和度（S）**：根据 L 是否大于 0.5 选择不同公式。
3. **色相（H）**：与 HSV 转换类似的六扇区计算。

#### hsl_to_rgb 算法
使用解析公式直接计算，基于色相的六扇区映射和色度（chroma）参数。

## 关键说明

源文件中包含 TODO 注释 `/* TODO(lukas): Fix colors in OSL. */`，表明 OSL 中的颜色处理可能存在已知的精度或行为问题，后续版本可能会修正。
