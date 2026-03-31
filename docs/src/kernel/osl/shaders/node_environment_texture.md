# node_environment_texture.osl - 环境纹理着色器

## 概述

该开放着色语言(OSL)着色器节点用于将环境贴图（HDR/EXR 等）映射到场景背景上。支持等距柱状投影（Equirectangular）和镜像球投影（Mirror Ball）两种映射方式，可处理 Alpha 通道及 sRGB 颜色空间转换，是场景照明和背景设置的核心纹理节点。

## 着色器签名

```osl
shader node_environment_texture(
    int use_mapping = 0,
    matrix mapping = matrix(0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0),
    vector Vector = P,
    string filename = "",
    string projection = "equirectangular",
    string interpolation = "linear",
    int compress_as_srgb = 0,
    int ignore_alpha = 0,
    int unassociate_alpha = 0,
    int is_float = 1,
    output color Color = 0.0,
    output float Alpha = 1.0)
```

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| use_mapping | int | 0 | 是否启用纹理坐标映射变换 |
| mapping | matrix | 零矩阵 | 纹理坐标变换矩阵 |
| Vector | vector | P | 输入方向向量 |
| filename | string | "" | 环境贴图文件路径 |
| projection | string | "equirectangular" | 投影方式："equirectangular"（等距柱状）或 "mirrorball"（镜像球） |
| interpolation | string | "linear" | 纹理插值方式 |
| compress_as_srgb | int | 0 | 是否将纹理从 sRGB 转换到场景线性空间 |
| ignore_alpha | int | 0 | 是否忽略 Alpha 通道 |
| unassociate_alpha | int | 0 | 是否对预乘 Alpha 进行反关联 |
| is_float | int | 1 | 纹理是否为浮点格式 |

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| Color | color | 采样得到的环境贴图颜色 |
| Alpha | float | 采样得到的 Alpha 透明度值 |

## 实现逻辑

1. **方向归一化**：将输入向量归一化为单位方向向量。
2. **投影映射**：
   - **等距柱状投影**：使用 `atan2` 和 `hypot` 将方向向量转换为 (u, v) 纹理坐标，u 对应水平角度，v 对应垂直仰角。
   - **镜像球投影**：模拟镜面球的反射映射，将方向向量投影到 [0,1] 的 UV 空间。
3. **纹理采样**：使用 OSL `texture()` 函数以 "periodic" 环绕模式读取贴图，v 坐标翻转（1.0 - v）以匹配纹理方向。
4. **Alpha 处理**：根据 `ignore_alpha` 和 `unassociate_alpha` 参数处理 Alpha 通道，对非浮点纹理限制颜色值在 [0,1] 范围。
5. **颜色空间转换**：若 `compress_as_srgb` 为真，将颜色从 sRGB 转换到场景线性空间。

## 对应 SVM 节点

对应 `svm_image` 相关实现（`src/kernel/svm/image.h`），环境纹理模式。
