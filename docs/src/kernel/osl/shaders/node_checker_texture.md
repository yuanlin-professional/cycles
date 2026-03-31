# node_checker_texture.osl - 棋盘格纹理着色器

## 概述

该开放着色语言(OSL)着色器节点生成三维棋盘格纹理图案。通过对三维空间坐标进行整数取模运算，生成交替排列的双色棋盘格，常用于测试 UV 映射、地板材质以及程序化图案生成。

## 着色器签名

```osl
shader node_checker_texture(
    int use_mapping = 0,
    matrix mapping = matrix(0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0),
    float Scale = 5.0,
    point Vector = P,
    color Color1 = 0.8,
    color Color2 = 0.2,
    output float Fac = 0.0,
    output color Color = 0.0)
```

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| use_mapping | int | 0 | 是否启用纹理坐标映射变换 |
| mapping | matrix | 零矩阵 | 纹理坐标变换矩阵 |
| Scale | float | 5.0 | 棋盘格缩放因子（值越大格子越密） |
| Vector | point | P | 输入纹理坐标 |
| Color1 | color | 0.8 | 棋盘格颜色 1 |
| Color2 | color | 0.2 | 棋盘格颜色 2 |

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| Fac | float | 棋盘格因子（1.0 = Color1 区域，0.0 = Color2 区域） |
| Color | color | 输出颜色（根据 Fac 选择 Color1 或 Color2） |

## 实现逻辑

1. **坐标预处理**：将输入坐标乘以 Scale，并添加微小偏移（0.000001）再乘以 0.999999 以避免精度问题导致的边界伪影。
2. **整数网格计算**：对 x、y、z 分量分别取 `floor` 后取绝对值，得到整数网格索引 `xi`、`yi`、`zi`。
3. **奇偶判定**：检查条件 `(xi % 2 == yi % 2) == (zi % 2)`，若为真返回 1.0（Color1），否则返回 0.0（Color2）。这实现了三维空间中的棋盘格交替排列。
4. **颜色选择**：根据 Fac 值选取对应颜色输出。

## 对应 SVM 节点

对应 `svm_checker` 相关实现（`src/kernel/svm/checker.h`）。
