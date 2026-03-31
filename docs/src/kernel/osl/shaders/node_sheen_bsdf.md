# node_sheen_bsdf.osl - 光泽绒面双向散射分布函数(BSDF)着色器

## 概述

该着色器实现了绒面/天鹅绒材质的光照计算，模拟织物表面在掠射角处产生的特有光泽效果。支持两种分布模型：Ashikhmin 天鹅绒模型和微纤维（Microfiber）模型。常用于模拟布料、天鹅绒、绒毛等材质的边缘高光。

## 着色器签名

```osl
shader node_sheen_bsdf(color Color = 0.8,
                       string distribution = "microfiber",
                       float Roughness = 0.0,
                       normal Normal = N,
                       output closure color BSDF = 0)
```

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| Color | color | 0.8 | 绒面光泽颜色，负值被钳制为 0 |
| distribution | string | "microfiber" | 分布模型：`"ashikhmin"`（Ashikhmin 天鹅绒）或 `"microfiber"`（微纤维） |
| Roughness | float | 0.0 | 粗糙度，范围 [0, 1] |
| Normal | normal | N | 表面法线方向 |

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| BSDF | closure color | 绒面光泽闭包(Closure)输出 |

## 实现逻辑

1. **颜色处理**：将颜色钳制为非负值。
2. **粗糙度处理**：钳制到 [0, 1] 范围。
3. **分布选择**：
   - `"ashikhmin"` 模式：使用 `ashikhmin_velvet(Normal, roughness)` 闭包，基于 Ashikhmin 天鹅绒反射模型。
   - `"microfiber"` 模式：使用 `sheen(Normal, roughness)` 闭包，基于微纤维模型，提供更物理精确的织物外观。

## 对应 SVM 节点

对应 SVM 中的 `NODE_CLOSURE_BSDF`，闭包类型为 `CLOSURE_BSDF_ASHIKHMIN_VELVET_ID` 和 `CLOSURE_BSDF_SHEEN_ID`。
