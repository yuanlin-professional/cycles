# node_layer_weight.osl - 层权重着色器

## 概述

层权重着色器（Layer Weight Node）用于生成基于观察角度的混合权重，常用于在两个材质层之间进行平滑过渡。该着色器是开放着色语言（OSL）中的输入类节点，提供菲涅耳和朝向两种权重模式。

## 着色器签名

```osl
shader node_layer_weight(
    float Blend = 0.5,
    normal Normal = N,
    output float Fresnel = 0.0,
    output float Facing = 0.0
)
```

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| Blend | float | `0.5` | 混合因子，控制菲涅耳效果的强度和朝向权重的衰减曲线 |
| Normal | normal | `N` | 用于计算的法线方向，默认为着色点的插值法线 |

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| Fresnel | float | 基于菲涅耳方程计算的权重，边缘区域值较大 |
| Facing | float | 基于朝向角度计算的权重，与观察方向越垂直值越大 |

## 实现逻辑

1. **菲涅耳权重**：
   - 根据 `Blend` 计算折射率 `eta = max(1.0 - Blend, 1e-5)`。
   - 根据是否为背面调整 eta 值：背面使用 `eta`，正面使用 `1.0/eta`。
   - 计算入射方向与法线的点积 `cosi = dot(I, Normal)`。
   - 调用 `fresnel_dielectric_cos(cosi, eta)` 计算介电质菲涅耳反射率。

2. **朝向权重**：
   - 初始值为入射方向与法线点积的绝对值：`Facing = fabs(cosi)`。
   - 当 `Blend` 不等于 0.5 时，应用自定义衰减曲线：
     - 将 `blend` 限制在 `[0, 1-1e-5]` 范围。
     - `blend < 0.5` 时：指数 = `2.0 * blend`。
     - `blend >= 0.5` 时：指数 = `0.5 / (1.0 - blend)`。
     - 计算 `Facing = pow(Facing, blend)`。
   - 最终取反：`Facing = 1.0 - Facing`，使得边缘区域值较大。

## 对应 SVM 节点

对应 Cycles SVM（着色器虚拟机）中的 `NODE_LAYER_WEIGHT` 节点。
