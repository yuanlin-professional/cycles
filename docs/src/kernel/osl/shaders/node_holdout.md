# node_holdout.osl - 遮罩着色器

## 概述

该着色器实现了遮罩（Holdout）效果。遮罩区域在渲染中会变为完全透明（Alpha 为 0），用于在渲染结果中创建"洞"。常用于合成工作流中，将 3D 场景与实拍素材结合时遮挡特定区域。

## 着色器签名

```osl
shader node_holdout(output closure color Holdout = holdout())
```

## 输入参数

无输入参数。

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| Holdout | closure color | 遮罩闭包(Closure)输出，默认值直接为 `holdout()` 闭包 |

## 实现逻辑

着色器体为空，输出参数的默认值直接设为开放着色语言(OSL)内置的 `holdout()` 闭包。该闭包使表面在最终渲染中产生零 Alpha 区域，不反射也不透射任何光线。

## 对应 SVM 节点

对应 SVM 中的 `NODE_CLOSURE_HOLDOUT`，闭包类型为 `CLOSURE_HOLDOUT_ID`。
