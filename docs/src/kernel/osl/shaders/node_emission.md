# node_emission.osl - 自发光着色器

## 概述

该着色器实现了表面自发光效果。使物体表面向外发射光线，可作为场景中的光源使用。发光强度由颜色和强度参数共同控制。

## 着色器签名

```osl
shader node_emission(color Color = 0.8,
                     float Strength = 1.0,
                     output closure color Emission = 0)
```

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| Color | color | 0.8 | 发光颜色 |
| Strength | float | 1.0 | 发光强度 |

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| Emission | closure color | 自发光闭包(Closure)输出 |

## 实现逻辑

计算公式为 `(Strength * Color) * emission()`。将强度和颜色相乘作为权重，乘以开放着色语言(OSL)内置的 `emission()` 闭包。

## 对应 SVM 节点

对应 SVM 中的 `NODE_CLOSURE_EMISSION`，闭包类型为 `CLOSURE_EMISSION_ID`。
