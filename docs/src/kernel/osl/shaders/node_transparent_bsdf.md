# node_transparent_bsdf.osl - 透明双向散射分布函数(BSDF)着色器

## 概述

该着色器实现了完全透明的表面。光线直接穿过表面而不发生散射或折射，仅受颜色权重影响。常用于实现 Alpha 透明度、阴影遮罩等效果。与折射着色器不同，透明着色器不改变光线方向。

## 着色器签名

```osl
shader node_transparent_bsdf(color Color = 0.8,
                             normal Normal = N,
                             output closure color BSDF = 0)
```

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| Color | color | 0.8 | 透明滤光颜色，白色为完全透明，黑色为完全不透明 |
| Normal | normal | N | 表面法线方向（透明闭包实际上不使用此参数） |

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| BSDF | closure color | 透明闭包(Closure)输出 |

## 实现逻辑

直接使用开放着色语言(OSL)内置的 `transparent()` 闭包，乘以颜色权重输出。该闭包使光线以原方向和强度穿过表面，不发生偏折。注意虽然签名中包含 Normal 参数，但 `transparent()` 闭包不使用法线。

## 对应 SVM 节点

对应 SVM 中的 `NODE_CLOSURE_BSDF`，闭包类型为 `CLOSURE_BSDF_TRANSPARENT_ID`。
