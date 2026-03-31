# node_background.osl - 背景着色器

## 概述

该着色器实现了世界环境背景光照。用于设置场景的环境光颜色和强度，为场景提供整体照明和背景可见性。通常连接到世界（World）材质输出。

## 着色器签名

```osl
shader node_background(color Color = 0.8,
                       float Strength = 1.0,
                       output closure color Background = 0)
```

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| Color | color | 0.8 | 背景颜色 |
| Strength | float | 1.0 | 背景光强度 |

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| Background | closure color | 背景闭包(Closure)输出 |

## 实现逻辑

计算公式为 `Color * Strength * background()`。将颜色和强度相乘作为权重，乘以开放着色语言(OSL)内置的 `background()` 闭包。

## 对应 SVM 节点

对应 SVM 中的 `NODE_CLOSURE_BACKGROUND`，闭包类型为 `CLOSURE_BACKGROUND_ID`。
