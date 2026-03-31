# node_magic_texture.osl - 魔法纹理着色器

## 概述

该开放着色语言(OSL)着色器节点生成彩色的程序化"魔法"纹理。通过对三角函数进行多层嵌套迭代，产生色彩丰富、形态复杂的抽象图案。迭代深度（depth）控制图案复杂度，扭曲度（distortion）控制变形强度，适用于生成迷幻效果和抽象艺术纹理。

## 着色器签名

```osl
shader node_magic_texture(
    int use_mapping = 0,
    matrix mapping = matrix(0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0),
    int depth = 2,
    float Distortion = 5.0,
    float Scale = 5.0,
    point Vector = P,
    output float Fac = 0.0,
    output color Color = 0.0)
```

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| use_mapping | int | 0 | 是否启用纹理坐标映射变换 |
| mapping | matrix | 零矩阵 | 纹理坐标变换矩阵 |
| depth | int | 2 | 迭代深度（0-10），越高图案越复杂 |
| Distortion | float | 5.0 | 扭曲强度 |
| Scale | float | 5.0 | 纹理缩放 |
| Vector | point | P | 输入纹理坐标 |

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| Fac | float | 颜色 RGB 三通道的平均值 |
| Color | color | 生成的彩色纹理值 |

## 实现逻辑

1. **初始角度**：将输入坐标各分量乘以 Scale 并对 2*PI 取模，得到初始角度 a、b、c。
2. **初始值**：
   - `x = sin((a + b + c) * 5.0)`
   - `y = cos((-a + b - c) * 5.0)`
   - `z = -cos((-a - b + c) * 5.0)`
3. **迭代变形**：根据 depth（0-10），依次对 x、y、z 执行交替的 `sin`/`cos` 运算，每步乘以 distortion。每一层迭代使用不同的符号组合和三角函数，形成递进式的变形效果。
4. **归一化**：最终将 x、y、z 除以 `2 * distortion` 并加 0.5 偏移，使输出中心化到 0.5 附近。
5. **Fac 计算**：输出颜色 RGB 三通道的算术平均值。

## 对应 SVM 节点

对应 `svm_magic` 相关实现（`src/kernel/svm/magic.h`）。
