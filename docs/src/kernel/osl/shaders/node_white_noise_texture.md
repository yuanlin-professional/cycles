# node_white_noise_texture.osl - 白噪声纹理着色器

## 概述

该开放着色语言(OSL)着色器节点生成白噪声纹理。白噪声的特点是完全随机，相邻像素间无相关性。支持 1D/2D/3D/4D 四个维度，使用哈希函数生成确定性伪随机值，适用于产生随机种子、随机颜色、ID 遮罩等需要均匀分布随机值的场景。

## 着色器签名

```osl
shader node_white_noise_texture(
    string dimensions = "3D",
    point Vector = point(0.0, 0.0, 0.0),
    float W = 0.0,
    output float Value = 0.0,
    output color Color = 0.0)
```

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| dimensions | string | "3D" | 噪声维度："1D"、"2D"、"3D"、"4D" |
| Vector | point | (0,0,0) | 输入坐标（用于 2D/3D/4D） |
| W | float | 0.0 | 标量输入（用于 1D 和 4D） |

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| Value | float | 随机标量值，范围 [0, 1] |
| Color | color | 随机颜色值，各通道独立在 [0, 1] 范围 |

## 实现逻辑

1. **维度分发**：根据 `dimensions` 参数选择计算路径。
2. **Value 计算**：使用 OSL 内置 `noise("hash", ...)` 函数生成单个浮点哈希值。
   - 1D：`noise("hash", W)`
   - 2D：`noise("hash", Vector[0], Vector[1])`
   - 3D：`noise("hash", Vector)`
   - 4D：`noise("hash", Vector, W)`
3. **Color 计算**：调用 `node_hash.h` 中的 `hash_*_to_color` 函数，通过对输入附加不同常数偏移进行多次哈希，生成三个独立的随机颜色通道。
   - 1D：`hash_float_to_color(W)`
   - 2D：`hash_vector2_to_color(vector2(Vector[0], Vector[1]))`
   - 3D：`hash_vector3_to_color(vector3(Vector[0], Vector[1], Vector[2]))`
   - 4D：`hash_vector4_to_color(vector4(Vector[0], Vector[1], Vector[2], W))`

## 对应 SVM 节点

对应 `svm_white_noise` 相关实现（`src/kernel/svm/white_noise.h`）。
