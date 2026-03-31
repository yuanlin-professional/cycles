# node_ambient_occlusion.osl - 环境光遮蔽着色器

## 概述

环境光遮蔽着色器（Ambient Occlusion Node）用于计算着色点的环境光遮蔽值。该着色器是开放着色语言（OSL）中的输入类节点，通过从着色点向周围发射采样光线来评估几何体的遮挡程度，遮蔽值越小表示该区域越被遮挡。

## 着色器签名

```osl
shader node_ambient_occlusion(
    color ColorIn = color(1.0, 1.0, 1.0),
    int samples = 16,
    float Distance = 1.0,
    normal Normal = N,
    int inside = 0,
    int only_local = 0,
    output color ColorOut = color(1.0, 1.0, 1.0),
    output float AO = 1.0
)
```

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| ColorIn | color | `(1.0, 1.0, 1.0)` | 输入颜色，将与 AO 值相乘输出 |
| samples | int | `16` | 采样数量，值越大结果越平滑但计算越慢 |
| Distance | float | `1.0` | 遮蔽检测的最大距离，0 表示使用全局半径 |
| Normal | normal | `N` | 用于采样的法线方向 |
| inside | int | `0` | 是否计算内部遮蔽（从表面内侧采样），1 为是 |
| only_local | int | `0` | 是否仅考虑局部遮蔽（只检测同一物体），1 为是 |

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| ColorOut | color | 输入颜色乘以 AO 值的结果 |
| AO | float | 环境光遮蔽值，取值范围 [0, 1]，1.0 表示无遮蔽 |

## 实现逻辑

1. **全局半径检测**：当 `Distance == 0.0` 且 `Distance` 端口未连接时，标记为使用全局半径。

2. **法线归一化**：对输入法线进行归一化处理。

3. **AO 计算**：通过特殊的 `texture("@ao", ...)` 调用实现，将 AO 计算参数编码为纹理查找参数：
   - 第一个参数：采样数
   - 第二个参数：距离
   - 第三至五个参数：归一化法线的 x、y、z 分量
   - 第六个参数：内部遮蔽标志
   - `"sblur"` 参数：仅本地标志
   - `"tblur"` 参数：全局半径标志

   这是一种利用纹理调用接口传递自定义参数的技巧（代码注释中标注为 "Abuse texture call"）。

4. **颜色输出**：`ColorOut = ColorIn * AO`，将输入颜色按遮蔽值缩放。

## 对应 SVM 节点

对应 Cycles SVM（着色器虚拟机）中的 `NODE_AO` 节点。
