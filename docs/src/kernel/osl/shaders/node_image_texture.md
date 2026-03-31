# node_image_texture.osl - 图像纹理着色器

## 概述

该开放着色语言(OSL)着色器节点用于将图像文件映射到物体表面。支持平面（Flat）、盒体（Box）、球形（Sphere）和管状（Tube）四种投影方式，以及多种插值和扩展模式。可处理 UDIM 瓦片纹理、Alpha 通道及 sRGB 颜色空间转换，是材质系统中最常用的纹理节点。

## 着色器签名

```osl
shader node_image_texture(
    int use_mapping = 0,
    matrix mapping = matrix(0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0),
    point Vector = P,
    string filename = "",
    string projection = "flat",
    string interpolation = "smartcubic",
    string extension = "periodic",
    float projection_blend = 0.0,
    int compress_as_srgb = 0,
    int ignore_alpha = 0,
    int unassociate_alpha = 0,
    int is_tiled = 0,
    int is_float = 1,
    output color Color = 0.0,
    output float Alpha = 1.0)
```

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| use_mapping | int | 0 | 是否启用纹理坐标映射变换 |
| mapping | matrix | 零矩阵 | 纹理坐标变换矩阵 |
| Vector | point | P | 输入纹理坐标 |
| filename | string | "" | 图像文件路径 |
| projection | string | "flat" | 投影方式："flat"、"box"、"sphere"、"tube" |
| interpolation | string | "smartcubic" | 插值方式（linear、closest、cubic、smartcubic） |
| extension | string | "periodic" | 纹理边界扩展模式（periodic、clamp、black） |
| projection_blend | float | 0.0 | 盒体投影时面之间的混合值 |
| compress_as_srgb | int | 0 | 是否将纹理从 sRGB 转换到场景线性空间 |
| ignore_alpha | int | 0 | 是否忽略 Alpha 通道 |
| unassociate_alpha | int | 0 | 是否对预乘 Alpha 进行反关联 |
| is_tiled | int | 0 | 是否为 UDIM 瓦片纹理 |
| is_float | int | 1 | 纹理是否为浮点格式 |

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| Color | color | 采样得到的纹理颜色 |
| Alpha | float | 采样得到的 Alpha 透明度值 |

## 实现逻辑

1. **坐标变换**：可选应用 mapping 矩阵变换输入坐标。
2. **投影映射**：
   - **Flat（平面）**：直接使用 (x, y) 作为 UV，翻转 V 坐标（支持 UDIM 瓦片）。
   - **Box（盒体）**：将物体空间法线转换为重心坐标权重，根据法线方向从三个轴向面分别采样纹理并按权重混合。通过 `projection_blend` 控制面之间的过渡平滑度。
   - **Sphere（球形）**：使用 `map_to_sphere` 将方向向量映射到球面 UV 坐标。
   - **Tube（管状）**：使用 `map_to_tube` 将方向向量映射到圆柱面 UV 坐标。
3. **纹理查找**：调用 `image_texture_lookup` 函数执行 OSL `texture()` 采样。
4. **Alpha 处理**：根据参数处理 Alpha 通道（忽略 / 反预乘），对非浮点纹理限制颜色在 [0,1]。
5. **颜色空间转换**：若 `compress_as_srgb` 为真，执行 sRGB 到线性空间的转换。

## 对应 SVM 节点

对应 `svm_image` 相关实现（`src/kernel/svm/image.h`）。
