# node_ies_light.osl - IES 光源纹理着色器

## 概述

该开放着色语言(OSL)着色器节点用于加载 IES（Illuminating Engineering Society）光源配光文件，将真实光源的光强分布应用于灯光着色。IES 文件描述了灯具在各个方向上的光强分布，通过将方向向量转换为垂直角和水平角并查询配光曲线纹理，实现物理准确的灯具照明效果。

## 着色器签名

```osl
shader node_ies_light(
    int use_mapping = 0,
    matrix mapping = matrix(0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0),
    string filename = "",
    float Strength = 1.0,
    point Vector = I,
    output float Fac = 0.0)
```

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| use_mapping | int | 0 | 是否启用纹理坐标映射变换 |
| mapping | matrix | 零矩阵 | 纹理坐标变换矩阵 |
| filename | string | "" | IES 配光文件路径 |
| Strength | float | 1.0 | 光源强度乘数 |
| Vector | point | I | 输入方向向量（默认为入射方向 I） |

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| Fac | float | 该方向上的光强因子（Strength * IES 查找值） |

## 实现逻辑

1. **方向归一化**：对输入向量（默认为入射方向 I）进行归一化处理。
2. **球面角度计算**：
   - **垂直角（v_angle）**：`acos(-p[2])`，从 Z 轴负方向测量的天顶角。
   - **水平角（h_angle）**：`atan2(p[0], p[1]) + PI`，在 XY 平面上的方位角，偏移 PI 使范围为 [0, 2*PI]。
3. **纹理查找**：使用水平角和垂直角作为 UV 坐标查询 IES 配光纹理文件。
4. **强度缩放**：将查找结果乘以 Strength 参数输出最终光强因子。

## 对应 SVM 节点

对应 `svm_ies` 相关实现（`src/kernel/svm/ies.h`）。
