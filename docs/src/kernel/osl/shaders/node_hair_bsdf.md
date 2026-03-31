# node_hair_bsdf.osl - 毛发双向散射分布函数(BSDF)着色器

## 概述

该着色器实现了基础的毛发光照计算，提供反射和透射两种分量。与原理化毛发着色器相比，这是一个更简单的毛发着色模型，直接使用 U/V 方向的粗糙度参数。支持曲线几何体和普通网格上的毛发渲染。

## 着色器签名

```osl
shader node_hair_bsdf(color Color = 0.8,
                      string component = "reflection",
                      float Offset = 0.0,
                      float RoughnessU = 0.1,
                      float RoughnessV = 1.0,
                      normal Tangent = normal(0, 0, 0),
                      output closure color BSDF = 0)
```

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| Color | color | 0.8 | 毛发颜色 |
| component | string | "reflection" | 散射分量：`"reflection"`（反射）或 `"transmission"`（透射） |
| Offset | float | 0.0 | 毛鳞片偏移角度，内部取负值 |
| RoughnessU | float | 0.1 | U 方向（纵向）粗糙度，范围 [0.001, 1.0] |
| RoughnessV | float | 1.0 | V 方向（径向）粗糙度，范围 [0.001, 1.0] |
| Tangent | normal | (0, 0, 0) | 切线方向，未连接时自动从几何体推导 |

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| BSDF | closure color | 毛发散射闭包(Closure)输出 |

## 实现逻辑

1. **粗糙度钳制**：将 RoughnessU 和 RoughnessV 钳制到 [0.001, 1.0] 范围。
2. **偏移处理**：Offset 取负值。
3. **切线确定**：
   - 若 Tangent 端口已连接，直接使用。
   - 若几何体不是曲线（`geom:is_curve` 为假），使用 `dPdv` 导数并将偏移归零。
   - 若几何体是曲线，使用 `dPdu` 导数作为切线。
4. **背面处理**：若光线从曲线背面入射，返回透明闭包。
5. **分量选择**：
   - `"reflection"`：使用 `hair_reflection(N, roughnessh, roughnessv, T, offset)` 闭包。
   - `"transmission"`：使用 `hair_transmission(N, roughnessh, roughnessv, T, offset)` 闭包。

## 对应 SVM 节点

对应 SVM 中的 `NODE_CLOSURE_BSDF`，闭包类型为 `CLOSURE_BSDF_HAIR_REFLECTION_ID` 和 `CLOSURE_BSDF_HAIR_TRANSMISSION_ID`。
