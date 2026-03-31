# node_add_closure.osl - 闭包相加着色器

## 概述

该着色器将两个闭包(Closure)进行相加操作。这是闭包混合的基本方式之一，用于叠加不同的着色效果。与混合闭包不同，相加闭包不会对能量进行归一化，两个输入闭包的贡献是直接叠加的。

## 着色器签名

```osl
shader node_add_closure(closure color Closure1 = 0,
                        closure color Closure2 = 0,
                        output closure color Closure = 0)
```

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| Closure1 | closure color | 0 | 第一个闭包(Closure)输入 |
| Closure2 | closure color | 0 | 第二个闭包(Closure)输入 |

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| Closure | closure color | 相加结果闭包(Closure)输出 |

## 实现逻辑

直接将两个输入闭包相加：`Closure = Closure1 + Closure2`。

注意：由于不进行能量守恒检查，若两个输入闭包的总能量超过 1，可能导致渲染结果过亮。用户需自行确保能量守恒。

## 对应 SVM 节点

对应 SVM 中的 `NODE_ADD_CLOSURE`。
