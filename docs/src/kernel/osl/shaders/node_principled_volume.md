# node_principled_volume.osl - 原理化体积着色器

## 概述

该着色器是 Cycles 渲染器中最综合的体积(Volume)着色器。将体积散射、吸收和自发光统一在单个着色器中，并支持从几何体属性（如 OpenVDB 体积数据）读取密度、颜色和温度信息。支持黑体辐射模拟，常用于烟雾、火焰等体积效果的渲染。

## 着色器签名

```osl
shader node_principled_volume(color Color = color(0.5, 0.5, 0.5),
                              float Density = 1.0,
                              float Anisotropy = 0.0,
                              color AbsorptionColor = color(0.0, 0.0, 0.0),
                              float EmissionStrength = 0.0,
                              color EmissionColor = color(1.0, 1.0, 1.0),
                              float BlackbodyIntensity = 0.0,
                              color BlackbodyTint = color(1.0, 1.0, 1.0),
                              float Temperature = 1500.0,
                              string DensityAttribute = "geom:density",
                              string ColorAttribute = "geom:color",
                              string TemperatureAttribute = "geom:temperature",
                              output closure color Volume = 0)
```

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| Color | color | (0.5, 0.5, 0.5) | 散射颜色，控制体积介质散射光的颜色 |
| Density | float | 1.0 | 基础密度，最小值为 0 |
| Anisotropy | float | 0.0 | 散射各向异性（Henyey-Greenstein 参数） |
| AbsorptionColor | color | (0.0, 0.0, 0.0) | 吸收颜色，黑色表示无额外吸收 |
| EmissionStrength | float | 0.0 | 自发光强度 |
| EmissionColor | color | (1.0, 1.0, 1.0) | 自发光颜色 |
| BlackbodyIntensity | float | 0.0 | 黑体辐射强度 |
| BlackbodyTint | color | (1.0, 1.0, 1.0) | 黑体辐射色调 |
| Temperature | float | 1500.0 | 基础温度（开尔文），用于黑体辐射 |
| DensityAttribute | string | "geom:density" | 密度属性名称 |
| ColorAttribute | string | "geom:color" | 颜色属性名称 |
| TemperatureAttribute | string | "geom:temperature" | 温度属性名称 |

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| Volume | closure color | 原理化体积闭包(Closure)输出 |

## 实现逻辑

### 1. 密度计算
- 基础密度钳制为非负值。
- 若密度 > `1e-5` 且存在密度属性（如 OpenVDB 的 density 通道），将基础密度与属性值相乘。

### 2. 散射与吸收
当密度 > `1e-5` 时：
- **散射颜色**：基础颜色与颜色属性（如 OpenVDB 的 color 通道）相乘。
- **散射系数**：等于散射颜色。
- **吸收系数**：`max(1 - scatter_color, 0) * max(1 - sqrt(AbsorptionColor), 0)`。
  - AbsorptionColor 取平方根以提供更感知线性的控制。
  - 吸收仅作用于散射颜色未覆盖的光谱部分。
- 体积闭包为 `scatter_coeff * density * henyey_greenstein(Anisotropy) + absorption_coeff * density * absorption()`。

### 3. 自发光
当 EmissionStrength > `1e-5` 时：
- 叠加 `emission_strength * EmissionColor * emission()` 闭包。

### 4. 黑体辐射
当 BlackbodyIntensity > `1e-3` 时：
- 温度 T 与温度属性（如 OpenVDB 的 temperature 通道）相乘。
- 根据 Stefan-Boltzmann 定律计算辐射强度：`sigma * mix(1.0, T^4, blackbody_intensity)`。
  - 常数 `sigma = 5.670373e-8 * 1e-6 / pi`。
- 使用 `blackbody(T)` 函数获取对应温度的光谱颜色。
- 归一化亮度后乘以 BlackbodyTint 和强度。
- 叠加到体积闭包中。

## 对应 SVM 节点

对应 SVM 中的 `NODE_PRINCIPLED_VOLUME`，内部组合 `CLOSURE_VOLUME_HENYEY_GREENSTEIN_ID` 和 `CLOSURE_VOLUME_ABSORPTION_ID` 以及 `CLOSURE_EMISSION_ID`。
