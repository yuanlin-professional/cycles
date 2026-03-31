# node_gabor_texture.osl - Gabor 噪声纹理着色器

## 概述

该开放着色语言(OSL)着色器节点实现了基于 Gabor 噪声的程序化纹理。基于 Lagae 等人 2009 年的稀疏 Gabor 卷积论文，并融合了 Tavernier 等人 2019 年的快速归一化改进以及 Tricard 等人 2019 年的 Phasor 噪声相位/强度计算。支持 2D 和 3D 两种维度模式，可控制频率、各向异性和方向，适用于生成规则条纹、波纹等方向性纹理图案。

## 着色器签名

```osl
shader node_gabor_texture(
    int use_mapping = 0,
    matrix mapping = matrix(0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0),
    string type = "2D",
    point Vector = P,
    float Scale = 5.0,
    float Frequency = 2.0,
    float Anisotropy = 1.0,
    float Orientation2D = M_PI / 4.0,
    point Orientation3D = point(M_SQRT2, M_SQRT2, 0.0),
    output float Value = 0.0,
    output float Phase = 0.0,
    output float Intensity = 0.0)
```

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| use_mapping | int | 0 | 是否启用纹理坐标映射变换 |
| mapping | matrix | 零矩阵 | 纹理坐标变换矩阵 |
| type | string | "2D" | 噪声维度类型："2D" 或 "3D" |
| Vector | point | P | 输入纹理坐标 |
| Scale | float | 5.0 | 纹理整体缩放 |
| Frequency | float | 2.0 | Gabor 核的频率参数（F_0），控制条纹密度 |
| Anisotropy | float | 1.0 | 各向异性程度（1.0 = 完全各向异性，0.0 = 各向同性） |
| Orientation2D | float | M_PI/4.0 | 2D 模式下的方向角度（弧度） |
| Orientation3D | point | (sqrt2, sqrt2, 0) | 3D 模式下的方向向量 |

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| Value | float | Gabor 噪声值，映射到 [0, 1] 范围 |
| Phase | float | Phasor 相位值，映射到 [0, 1] 范围 |
| Intensity | float | Phasor 强度值（振幅） |

## 实现逻辑

1. **坐标缩放**：输入坐标乘以 Scale，计算各向同性度 `isotropy = 1.0 - Anisotropy`。
2. **Gabor 核计算**：每个网格单元内放置 8 个脉冲（`IMPULSES_COUNT = 8`），每个脉冲由高斯包络乘以 Hann 窗函数再乘以正弦/余弦相量构成。
3. **网格遍历**：在当前单元及其 3x3（2D）或 3x3x3（3D）邻域中累加所有 Gabor 核贡献，权重基于伯努利分布（+1 或 -1）。
4. **方向控制**：各向异性噪声使用固定方向，各向同性噪声在每个脉冲上随机旋转方向，通过 isotropy 参数线性插值。
5. **归一化**：使用 6 倍标准差进行经验归一化。标准差基于 Gabor 核平方积分和脉冲密度计算。
6. **输出计算**：
   - **Value**：取相量虚部（正弦部分），映射到 [0, 1]。
   - **Phase**：使用 `atan2` 计算相量角度，映射到 [0, 1]。
   - **Intensity**：计算相量长度（振幅）。

## 对应 SVM 节点

对应 `svm_gabor_texture` 相关实现（`src/kernel/svm/gabor.h`）。
