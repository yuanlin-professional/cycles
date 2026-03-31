# node_light_falloff.osl - 灯光衰减着色器

## 概述

该着色器计算灯光随距离的衰减方式，提供三种衰减模式：二次衰减（Quadratic）、线性衰减（Linear）和常数衰减（Constant）。同时支持平滑过渡参数，用于避免灯光近距离处的过度高亮。

## 着色器签名

```osl
shader node_light_falloff(float Strength = 0.0,
                          float Smooth = 0.0,
                          output float Quadratic = 0.0,
                          output float Linear = 0.0,
                          output float Constant = 0.0)
```

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| Strength | float | 0.0 | 灯光强度基准值 |
| Smooth | float | 0.0 | 平滑系数，用于在近距离处平滑衰减曲线，避免强度趋于无穷大。0.0 表示不平滑 |

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| Quadratic | float | 二次衰减（物理真实），强度与距离平方成反比 |
| Linear | float | 线性衰减，强度与距离成反比 |
| Constant | float | 常数衰减，强度不随距离变化 |

## 实现逻辑

1. 通过 `getattribute("path:ray_length", ray_length)` 获取当前光线路径长度。
2. **远距离灯光特殊处理**：如果 `ray_length == FLT_MAX`（远距离灯光），三个输出均直接等于 `Strength`，避免溢出。
3. **平滑处理**（当 `Smooth > 0.0`）：
   - 计算 `squared = ray_length * ray_length`
   - 应用平滑公式：`strength *= squared / (Smooth + squared)`
   - 该公式在距离趋近于零时使强度趋向零，而不是无穷大。
4. **三种衰减输出**：
   - `Quadratic = strength`（已包含 1/r^2 物理衰减）
   - `Linear = strength * ray_length`（补偿一个距离因子，得到 1/r 衰减）
   - `Constant = strength * ray_length * ray_length`（补偿两个距离因子，无衰减）

## 对应 SVM 节点

对应 Cycles SVM 中的 `NODE_LIGHT_FALLOFF` 节点。该节点在 Blender 节点编辑器中显示为"灯光衰减"（Light Falloff）节点，通常连接到灯光着色器的强度输入。
