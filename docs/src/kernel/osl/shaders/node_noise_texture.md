# node_noise_texture.osl - 噪声纹理着色器

## 概述

该开放着色语言(OSL)着色器节点生成多种类型的程序化噪声纹理。支持 1D/2D/3D/4D 四个维度以及五种噪声类型（fBM、多重分形、混合多重分形、脊状多重分形、异质地形），是 Cycles 渲染器中功能最全面的噪声生成节点。通过细节层级、粗糙度、缺陷度等参数精细控制噪声形态，可用于模拟自然材质、地形、云层等效果。

## 着色器签名

```osl
shader node_noise_texture(
    int use_mapping = 0,
    matrix mapping = matrix(0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0),
    string dimensions = "3D",
    string type = "fBM",
    int use_normalize = 1,
    point Vector = point(0, 0, 0),
    float W = 0.0,
    float Scale = 5.0,
    float Detail = 2.0,
    float Roughness = 0.5,
    float Offset = 0.0,
    float Gain = 1.0,
    float Lacunarity = 2.0,
    float Distortion = 0.0,
    output float Fac = 0.0,
    output color Color = 0.0)
```

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| use_mapping | int | 0 | 是否启用纹理坐标映射变换 |
| mapping | matrix | 零矩阵 | 纹理坐标变换矩阵 |
| dimensions | string | "3D" | 噪声维度："1D"、"2D"、"3D"、"4D" |
| type | string | "fBM" | 噪声类型（见下方说明） |
| use_normalize | int | 1 | 是否将输出归一化到 [0, 1] |
| Vector | point | (0,0,0) | 输入纹理坐标 |
| W | float | 0.0 | 第四维坐标（用于 1D 和 4D 模式） |
| Scale | float | 5.0 | 纹理缩放 |
| Detail | float | 2.0 | 细节层级（0-15），控制分形迭代次数 |
| Roughness | float | 0.5 | 粗糙度，控制各层级振幅衰减 |
| Offset | float | 0.0 | 偏移量（用于部分噪声类型） |
| Gain | float | 1.0 | 增益（用于混合/脊状多重分形） |
| Lacunarity | float | 2.0 | 缺陷度，控制各层级频率倍增 |
| Distortion | float | 0.0 | 畸变强度，对坐标进行噪声扰动 |

**type 可选值：**
- `"fBM"` - 分形布朗运动，标准分形噪声
- `"multifractal"` - 多重分形，各层级相乘
- `"hybrid_multifractal"` - 混合多重分形，结合加法与乘法
- `"ridged_multifractal"` - 脊状多重分形，产生脊线特征
- `"hetero_terrain"` - 异质地形，适合地形生成

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| Fac | float | 噪声值（归一化后通常在 [0, 1] 范围） |
| Color | color | 彩色噪声输出（使用不同偏移的噪声作为 RGB 三通道） |

## 实现逻辑

1. **坐标准备**：输入坐标乘以 Scale，W 维度同样乘以 Scale。
2. **畸变处理**：当 `Distortion != 0` 时，使用独立的 `safe_snoise` 对坐标各分量进行噪声偏移，产生扭曲效果。随机偏移函数生成 [100, 200] 范围内的偏移以充当种子。
3. **噪声选择**：通过 `noise_select` 宏分发到对应噪声类型函数（`noise_fbm`、`noise_multi_fractal` 等），所有噪声类型均定义在 `node_noise.h` 中。
4. **颜色生成**：对噪声坐标分别添加不同的随机偏移，生成三个独立的噪声值作为 RGB 三通道。
5. **维度分发**：根据 `dimensions` 参数选择 1D（float）、2D（vector2）、3D（vector3）或 4D（vector4）计算路径。

## 对应 SVM 节点

对应 `svm_noise` 相关实现（`src/kernel/svm/noise.h`）。
