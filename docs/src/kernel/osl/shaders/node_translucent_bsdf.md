# node_translucent_bsdf.osl - 半透明双向散射分布函数(BSDF)着色器

## 概述

该着色器实现了半透明表面的光照计算。半透明效果模拟光线穿过薄壁材质（如纸张、树叶等）时在背面产生的漫反射照明。与漫反射着色器类似，但光线从表面背面散射出去。

## 着色器签名

```osl
shader node_translucent_bsdf(color Color = 0.8,
                             normal Normal = N,
                             output closure color BSDF = 0)
```

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| Color | color | 0.8 | 半透明颜色 |
| Normal | normal | N | 表面法线方向 |

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| BSDF | closure color | 半透明闭包(Closure)输出 |

## 实现逻辑

直接使用开放着色语言(OSL)内置的 `translucent(Normal)` 闭包，乘以颜色权重输出。该闭包在法线反方向上进行 Lambert 漫反射采样。

## 对应 SVM 节点

对应 SVM 中的 `NODE_CLOSURE_BSDF`，闭包类型为 `CLOSURE_BSDF_TRANSLUCENT_ID`。
