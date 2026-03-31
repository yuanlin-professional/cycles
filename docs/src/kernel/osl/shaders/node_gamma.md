# node_gamma.osl - 伽马校正着色器

## 概述

该着色器对输入颜色执行伽马校正（Gamma Correction）运算。伽马校正是图像处理中的基本操作，通过幂函数调整颜色的明暗分布，常用于色调映射和颜色空间转换。

## 着色器签名

```osl
shader node_gamma(color ColorIn = 0.8,
                  float Gamma = 1.0,
                  output color ColorOut = 0.0)
```

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| ColorIn | color | 0.8 | 输入颜色 |
| Gamma | float | 1.0 | 伽马值。1.0 为无变化；小于 1.0 提亮中间调；大于 1.0 压暗中间调 |

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| ColorOut | color | 伽马校正后的输出颜色 |

## 实现逻辑

对输入颜色的每个通道执行幂运算：

```
ColorOut = pow(ColorIn, Gamma)
```

即 `ColorOut[i] = ColorIn[i] ^ Gamma`。

- 当 `Gamma = 1.0` 时，输出等于输入（恒等变换）。
- 当 `Gamma < 1.0` 时，中间调被提亮，整体图像变亮。
- 当 `Gamma > 1.0` 时，中间调被压暗，整体图像变暗。

开放着色语言（OSL）内置的 `pow()` 函数会自动处理颜色类型的逐通道运算。

## 对应 SVM 节点

对应 Cycles SVM 中的 `NODE_GAMMA` 节点。该节点在 Blender 节点编辑器中显示为"伽玛"（Gamma）节点。
