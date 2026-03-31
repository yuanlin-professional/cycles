# node_diffuse_bsdf.osl - 漫反射双向散射分布函数(BSDF)着色器

## 概述

该着色器实现了漫反射表面的光照计算。根据粗糙度参数的不同，着色器会在理想 Lambert 漫反射模型和 Oren-Nayar 漫反射模型之间自动切换。当粗糙度接近零时使用 Lambert 模型，当粗糙度有明显值时使用 Oren-Nayar 模型以模拟更真实的粗糙漫反射表面效果。

## 着色器签名

```osl
shader node_diffuse_bsdf(color Color = 0.8,
                         float Roughness = 0.0,
                         normal Normal = N,
                         output closure color BSDF = 0)
```

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| Color | color | 0.8 | 漫反射表面颜色 |
| Roughness | float | 0.0 | 表面粗糙度，控制漫反射的扩散程度。值为 0 时为理想 Lambert 漫反射 |
| Normal | normal | N | 表面法线方向，默认使用着色点法线 |

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| BSDF | closure color | 漫反射闭包(Closure)输出 |

## 实现逻辑

1. 判断粗糙度是否小于 `1e-5`（近似为零的阈值）。
2. 若粗糙度近似为零，使用 Lambert 漫反射模型：`Color * diffuse(Normal)`。
3. 若粗糙度有明显值，使用 Oren-Nayar 漫反射双向散射分布函数(BSDF)：`oren_nayar_diffuse_bsdf(Normal, clamp(Color, 0.0, 1.0), Roughness)`。其中颜色值被钳制在 `[0, 1]` 范围内。

## 对应 SVM 节点

对应 SVM 中的 `NODE_CLOSURE_BSDF`，闭包类型为 `CLOSURE_BSDF_DIFFUSE_ID`（Lambert）和 `CLOSURE_BSDF_OREN_NAYAR_ID`（Oren-Nayar）。
