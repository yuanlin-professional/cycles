# node_rgb_curves.osl - RGB 曲线着色器

## 概述

该着色器通过预定义的颜色渐变（Ramp）曲线对输入颜色的 R、G、B 三个通道分别进行非线性映射。每个颜色通道独立查找各自的曲线值，实现精细的色调调整。支持外推功能和混合因子控制。依赖 `node_ramp_util.h` 头文件中的 `rgb_ramp_lookup()` 函数。

## 着色器签名

```osl
shader node_rgb_curves(color ramp[] = {0.0},
                       float min_x = 0.0,
                       float max_x = 1.0,
                       int extrapolate = 1,
                       color ColorIn = 0.0,
                       float Fac = 0.0,
                       output color ColorOut = 0.0)
```

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| ramp | color[] | {0.0} | 曲线查找表数组，预计算的颜色渐变数据 |
| min_x | float | 0.0 | 曲线输入范围最小值 |
| max_x | float | 1.0 | 曲线输入范围最大值 |
| extrapolate | int | 1 | 是否启用外推。1 = 允许超出 [0, 1] 范围的外推 |
| ColorIn | color | 0.0 | 输入颜色 |
| Fac | float | 0.0 | 混合因子。0.0 = 输出原始颜色，1.0 = 完全应用曲线 |

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| ColorOut | color | 经过曲线映射并混合后的输出颜色 |

## 实现逻辑

1. **归一化输入**：将输入颜色各通道从 `[min_x, max_x]` 归一化到 `[0, 1]`：
   - `c = (ColorIn - min_x) / (max_x - min_x)`

2. **独立通道曲线查找**：每个通道使用自身的归一化值查找曲线：
   - `r = rgb_ramp_lookup(ramp, c[0], 1, extrapolate)` -> 用 R 通道值查找
   - `g = rgb_ramp_lookup(ramp, c[1], 1, extrapolate)` -> 用 G 通道值查找
   - `b = rgb_ramp_lookup(ramp, c[2], 1, extrapolate)` -> 用 B 通道值查找

3. **提取对应通道**：
   - `ColorOut[0] = r[0]`（R 查找结果的 R 分量）
   - `ColorOut[1] = g[1]`（G 查找结果的 G 分量）
   - `ColorOut[2] = b[2]`（B 查找结果的 B 分量）

4. **混合**：`ColorOut = mix(ColorIn, ColorOut, Fac)`。

与 `node_vector_curves.osl` 的关键区别在于：RGB 曲线的每个通道使用各自的输入值查找（`c[0]`、`c[1]`、`c[2]`），而向量曲线使用相同的输入值。

## 对应 SVM 节点

对应 Cycles SVM 中的 `NODE_RGB_CURVES` 节点。该节点在 Blender 节点编辑器中显示为"RGB 曲线"（RGB Curves）节点。
