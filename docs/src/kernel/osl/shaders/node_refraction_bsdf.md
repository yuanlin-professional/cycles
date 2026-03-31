# node_refraction_bsdf.osl - 折射双向散射分布函数(BSDF)着色器

## 概述

该着色器实现了纯折射（透射）表面的光照计算。与玻璃着色器不同，折射着色器仅计算透射光路而不包含反射分量。基于微表面模型，支持粗糙折射效果。着色器根据光线入射方向（正面或背面）自动调整折射率。

## 着色器签名

```osl
shader node_refraction_bsdf(color Color = 0.8,
                            string distribution = "ggx",
                            float Roughness = 0.2,
                            float IOR = 1.45,
                            normal Normal = N,
                            output closure color BSDF = 0)
```

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| Color | color | 0.8 | 折射颜色 |
| distribution | string | "ggx" | 微表面分布模型类型 |
| Roughness | float | 0.2 | 表面粗糙度，内部进行平方映射 |
| IOR | float | 1.45 | 折射率，最小值为 1e-5 |
| Normal | normal | N | 表面法线方向 |

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| BSDF | closure color | 折射闭包(Closure)输出 |

## 实现逻辑

1. **折射率处理**：IOR 最小值限制为 `1e-5`。若光线从背面入射（`backfacing()`），取倒数以正确处理从介质内部射出的情况。
2. **粗糙度映射**：进行平方映射（`Roughness * Roughness`）。
3. **闭包构建**：使用 `microfacet` 闭包，最后一个参数为 `1`，表示仅折射模式（refraction only）。

## 对应 SVM 节点

对应 SVM 中的 `NODE_CLOSURE_BSDF`，闭包类型为 `CLOSURE_BSDF_MICROFACET_GGX_REFRACTION_ID`。
