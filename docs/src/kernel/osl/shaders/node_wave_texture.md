# node_wave_texture.osl - 波纹纹理着色器

## 概述

该开放着色语言(OSL)着色器节点生成程序化波纹纹理。支持条带（Bands）和环形（Rings）两种波纹类型，以及正弦波（Sine）、锯齿波（Saw）和三角波（Tri）三种波形轮廓。可叠加 fBM 噪声进行畸变，适用于木纹、水波、年轮等自然纹理的模拟。

## 着色器签名

```osl
shader node_wave_texture(
    int use_mapping = 0,
    matrix mapping = matrix(0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0),
    string wave_type = "bands",
    string bands_direction = "x",
    string rings_direction = "x",
    string profile = "sine",
    float Scale = 5.0,
    float Distortion = 0.0,
    float Detail = 2.0,
    float DetailScale = 1.0,
    float DetailRoughness = 0.5,
    float PhaseOffset = 0.0,
    point Vector = P,
    output float Fac = 0.0,
    output color Color = 0.0)
```

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| use_mapping | int | 0 | 是否启用纹理坐标映射变换 |
| mapping | matrix | 零矩阵 | 纹理坐标变换矩阵 |
| wave_type | string | "bands" | 波纹类型："bands"（条带）或 "rings"（环形） |
| bands_direction | string | "x" | 条带方向："x"、"y"、"z"、"diagonal"（对角线） |
| rings_direction | string | "x" | 环形方向："x"、"y"、"z"、"spherical"（球形） |
| profile | string | "sine" | 波形轮廓："sine"（正弦）、"saw"（锯齿）、"tri"（三角） |
| Scale | float | 5.0 | 纹理缩放 |
| Distortion | float | 0.0 | 噪声畸变强度 |
| Detail | float | 2.0 | 畸变噪声的 fBM 细节层级 |
| DetailScale | float | 1.0 | 畸变噪声的缩放比例 |
| DetailRoughness | float | 0.5 | 畸变噪声的粗糙度 |
| PhaseOffset | float | 0.0 | 波形相位偏移 |
| Vector | point | P | 输入纹理坐标 |

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| Fac | float | 波纹值，范围 [0, 1] |
| Color | color | 灰度颜色输出（RGB 三通道均为 Fac 值） |

## 实现逻辑

1. **坐标预处理**：输入坐标乘以 Scale，添加微小偏移避免精度问题。
2. **基础波形计算**：
   - **条带模式**：根据 bands_direction 选择轴向分量乘以 20.0 作为基础值；对角线模式取 `(x + y + z) * 10.0`。
   - **环形模式**：将指定方向的分量置零后取向量长度乘以 20.0；球形模式保留所有分量。
3. **相位偏移**：加入 PhaseOffset。
4. **噪声畸变**：当 `Distortion != 0` 时，使用 `noise_fbm`（缺陷度 = 2.0）生成 fBM 噪声，将噪声值映射到 [-1, 1] 后乘以 Distortion 叠加到波形值上。
5. **波形轮廓**：
   - **Sine**：`0.5 + 0.5 * sin(n - PI/2)`
   - **Saw**：`n / (2*PI) - floor(n / (2*PI))`（锯齿函数）
   - **Tri**：`|n / (2*PI) - floor(n / (2*PI) + 0.5)| * 2.0`（三角函数）

## 对应 SVM 节点

对应 `svm_wave` 相关实现（`src/kernel/svm/wave.h`）。
