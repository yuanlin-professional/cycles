# node_mix_closure.osl - 闭包混合着色器

## 概述

该着色器根据混合因子在两个闭包(Closure)之间进行线性插值混合。这是闭包混合的标准方式，保证能量守恒。混合因子为 0 时完全使用第一个闭包，为 1 时完全使用第二个闭包。

## 着色器签名

```osl
shader node_mix_closure(float Fac = 0.5,
                        closure color Closure1 = 0,
                        closure color Closure2 = 0,
                        output closure color Closure = 0)
```

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| Fac | float | 0.5 | 混合因子，范围 [0, 1]，自动钳制 |
| Closure1 | closure color | 0 | 第一个闭包(Closure)输入 |
| Closure2 | closure color | 0 | 第二个闭包(Closure)输入 |

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| Closure | closure color | 混合结果闭包(Closure)输出 |

## 实现逻辑

1. 将混合因子钳制到 [0, 1] 范围：`t = clamp(Fac, 0.0, 1.0)`。
2. 进行线性插值：`Closure = (1.0 - t) * Closure1 + t * Closure2`。

该混合方式天然保证能量守恒，因为两个权重之和始终为 1。

## 对应 SVM 节点

对应 SVM 中的 `NODE_MIX_CLOSURE`。
