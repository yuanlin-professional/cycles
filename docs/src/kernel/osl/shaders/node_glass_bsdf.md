# node_glass_bsdf.osl - 玻璃双向散射分布函数(BSDF)着色器

## 概述

该着色器实现了玻璃材质的光照计算，同时包含反射和折射两种光路。基于广义 Schlick 双向散射分布函数(BSDF)模型，根据折射率（IOR）自动计算菲涅耳反射系数，并支持薄膜干涉效果。着色器会根据光线是否从背面进入来自动翻转折射率。

## 着色器签名

```osl
shader node_glass_bsdf(color Color = 0.8,
                       string distribution = "ggx",
                       float Roughness = 0.2,
                       float IOR = 1.45,
                       float ThinFilmThickness = 0.0,
                       float ThinFilmIOR = 1.33,
                       normal Normal = N,
                       output closure color BSDF = 0)
```

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| Color | color | 0.8 | 玻璃颜色，同时作用于反射和透射光，负值被钳制为 0 |
| distribution | string | "ggx" | 微表面分布模型类型 |
| Roughness | float | 0.2 | 表面粗糙度，范围 [0, 1]，内部进行平方映射 |
| IOR | float | 1.45 | 折射率（Index of Refraction），最小值为 1e-5 |
| ThinFilmThickness | float | 0.0 | 薄膜厚度，为 0 时无薄膜干涉效果 |
| ThinFilmIOR | float | 1.33 | 薄膜折射率 |
| Normal | normal | N | 表面法线方向 |

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| BSDF | closure color | 玻璃闭包(Closure)输出，包含反射和折射 |

## 实现逻辑

1. **颜色处理**：将颜色钳制为非负值。
2. **粗糙度映射**：钳制到 [0, 1] 后平方映射。
3. **折射率处理**：
   - IOR 最小值限制为 `1e-5`。
   - 若光线从背面入射（`backfacing()`），将薄膜 IOR 除以 eta，并将 eta 取倒数以正确处理从介质内部射出的情况。
4. **菲涅耳系数计算**：通过 `F0_from_ior(eta)` 计算法线入射时的反射率 F0，F90 设为 1.0。
5. **闭包构建**：使用 `generalized_schlick_bsdf` 闭包，eta 取负值表示同时启用反射和折射。传入薄膜参数以支持薄膜干涉。

## 对应 SVM 节点

对应 SVM 中的 `NODE_CLOSURE_BSDF`，闭包类型为 `CLOSURE_BSDF_MICROFACET_GGX_GLASS_ID`。
