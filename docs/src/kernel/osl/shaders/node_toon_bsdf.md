# node_toon_bsdf.osl - 卡通双向散射分布函数(BSDF)着色器

## 概述

该着色器实现了非真实感渲染（NPR）的卡通风格着色。支持漫反射卡通和光泽卡通两种模式，通过控制阶跃大小和平滑度来实现卡通渲染中特有的硬边光照过渡效果。

## 着色器签名

```osl
shader node_toon_bsdf(color Color = 0.8,
                      string component = "diffuse",
                      float Size = 0.5,
                      float Smooth = 0.0,
                      normal Normal = N,
                      output closure color BSDF = 0)
```

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| Color | color | 0.8 | 卡通着色颜色 |
| component | string | "diffuse" | 卡通着色模式：`"diffuse"`（漫反射卡通）或 `"glossy"`（光泽卡通） |
| Size | float | 0.5 | 明暗分界线的角度大小，控制受光区域范围 |
| Smooth | float | 0.0 | 明暗过渡的平滑程度，0 为硬边过渡 |
| Normal | normal | N | 表面法线方向 |

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| BSDF | closure color | 卡通着色闭包(Closure)输出 |

## 实现逻辑

根据 `component` 参数选择不同的卡通着色闭包：
- `"diffuse"` 模式：使用 `diffuse_toon(Normal, Size, Smooth)` 闭包，模拟卡通风格的漫反射。
- `"glossy"` 模式：使用 `glossy_toon(Normal, Size, Smooth)` 闭包，模拟卡通风格的高光反射。

两种模式均通过 Size 控制光照覆盖角度范围，Smooth 控制明暗交界处的渐变宽度。

## 对应 SVM 节点

对应 SVM 中的 `NODE_CLOSURE_BSDF`，闭包类型为 `CLOSURE_BSDF_DIFFUSE_TOON_ID` 和 `CLOSURE_BSDF_GLOSSY_TOON_ID`。
