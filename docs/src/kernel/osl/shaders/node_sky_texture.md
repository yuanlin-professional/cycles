# node_sky_texture.osl - 天空纹理着色器

## 概述

该开放着色语言(OSL)着色器节点生成物理模拟的天空纹理。支持三种天空模型：Preetham（1999）、Hosek/Wilkie（2012）和 Nishita（改进版多重散射）。通过模拟大气散射，可生成逼真的日间天空、日落效果和太阳盘，是场景环境照明的物理天空着色器。

## 着色器签名

```osl
shader node_sky_texture(
    int use_mapping = 0,
    matrix mapping = matrix(0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0),
    vector Vector = P,
    string sky_type = "multiple_scattering",
    float theta = 0.0,
    float phi = 0.0,
    string filename = "",
    color radiance = color(0.0, 0.0, 0.0),
    float config_x[9] = {0.0, ...},
    float config_y[9] = {0.0, ...},
    float config_z[9] = {0.0, ...},
    float nishita_data[11] = {0.0, ...},
    output color Color = color(0.0, 0.0, 0.0))
```

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| use_mapping | int | 0 | 是否启用纹理坐标映射变换 |
| mapping | matrix | 零矩阵 | 纹理坐标变换矩阵 |
| Vector | vector | P | 输入方向向量 |
| sky_type | string | "multiple_scattering" | 天空模型类型 |
| theta | float | 0.0 | 太阳天顶角（弧度） |
| phi | float | 0.0 | 太阳方位角（弧度） |
| filename | string | "" | Nishita 模型使用的天空 LUT 纹理路径 |
| radiance | color | (0,0,0) | Preetham/Hosek 模型的辐射度参数 |
| config_x[9] | float[] | 全零 | X 通道 Perez/Hosek 配置参数（9 个系数） |
| config_y[9] | float[] | 全零 | Y 通道 Perez/Hosek 配置参数 |
| config_z[9] | float[] | 全零 | Z 通道 Perez/Hosek 配置参数 |
| nishita_data[11] | float[] | 全零 | Nishita 模型数据（太阳盘像素、仰角、旋转、角直径等） |

**sky_type 可选值：**
- `"preetham"` - Preetham 天空模型，使用 Perez 函数
- `"hosek_wilkie"` - Hosek/Wilkie 天空模型，更精确的大气散射
- `"multiple_scattering"` / 其他 - Nishita 改进模型，支持太阳盘渲染和多重散射

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| Color | color | 天空辐射度颜色（RGB） |

## 实现逻辑

1. **方向计算**：将输入向量转换为球面坐标（天顶角 theta、方位角 phi），并计算与太阳方向的夹角 gamma。
2. **Preetham 模型**：使用 Perez 函数 `(1 + A*exp(B/cos(theta))) * (1 + C*exp(D*gamma) + E*cos^2(gamma))` 计算 xyY 颜色空间值，再转换为 RGB。
3. **Hosek/Wilkie 模型**：类似 Preetham 但使用更复杂的 9 参数辐射度函数，包含 Rayleigh 散射项、Mie 散射项和天顶项，结果乘以 `2*PI/683` 进行强度校正。
4. **Nishita 模型**：
   - **太阳盘**：检查射线是否在太阳角直径内，若是则从预计算像素数据插值并应用临边暗化效果（系数 0.6）。
   - **天空背景**：从 LUT 纹理采样 XYZ 颜色并转换为 RGB。方向仰角使用非线性映射以提升地平线附近精度。
   - 最终输出为太阳盘贡献（乘以太阳强度）与天空背景之和。

## 对应 SVM 节点

对应 `svm_sky` 相关实现（`src/kernel/svm/sky.h`）。
