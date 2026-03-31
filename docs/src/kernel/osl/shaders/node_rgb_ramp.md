# node_rgb_ramp.osl - RGB 渐变着色器

## 概述

该着色器通过因子值在预定义的颜色渐变（Color Ramp）中查找对应的颜色和透明度值。常用于将标量值映射为颜色，实现色带效果、色调映射和自定义颜色渐变。依赖 `node_ramp_util.h` 头文件中的 `rgb_ramp_lookup()` 函数。

## 着色器签名

```osl
shader node_rgb_ramp(color ramp_color[] = {0.0},
                     float ramp_alpha[] = {0.0},
                     int interpolate = 1,
                     float Fac = 0.0,
                     output color Color = 0.0,
                     output float Alpha = 1.0)
```

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| ramp_color | color[] | {0.0} | 颜色渐变查找表数组 |
| ramp_alpha | float[] | {0.0} | 透明度渐变查找表数组 |
| interpolate | int | 1 | 是否在采样点之间插值。1 = 线性插值，0 = 最近邻 |
| Fac | float | 0.0 | 查找因子，范围 [0, 1]，用于在渐变中定位 |

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| Color | color | 在渐变中查找到的颜色值 |
| Alpha | float | 在渐变中查找到的透明度值 |

## 实现逻辑

1. **颜色查找**：`Color = rgb_ramp_lookup(ramp_color, Fac, interpolate, 0)`。
2. **透明度查找**：`Alpha = rgb_ramp_lookup(ramp_alpha, Fac, interpolate, 0)`。

两者均调用 `rgb_ramp_lookup()` 函数，该函数：
- 将 `Fac` 钳制到 [0, 1] 范围（此处外推参数为 0，即禁用外推）。
- 根据 `Fac` 在渐变表中定位索引和插值系数。
- `interpolate = 1` 时在相邻采样点间线性插值，`interpolate = 0` 时取最近邻值。

## 对应 SVM 节点

对应 Cycles SVM 中的 `NODE_RGB_RAMP` 节点。该节点在 Blender 节点编辑器中显示为"颜色渐变"（ColorRamp）节点。
