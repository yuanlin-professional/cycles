# node_texture_coordinate.osl - 纹理坐标着色器

## 概述

纹理坐标着色器（Texture Coordinate Node）用于生成多种纹理映射坐标系统。该着色器是开放着色语言（OSL）中最重要的输入类节点之一，提供生成坐标、UV 坐标、物体坐标、相机坐标、窗口坐标、法线和反射方向等七种纹理坐标输出，是纹理采样的核心数据源。

## 着色器签名

```osl
shader node_texture_coordinate(
    int is_background = 0,
    int is_volume = 0,
    int from_dupli = 0,
    int use_transform = 0,
    string bump_offset = "center",
    float bump_filter_width = BUMP_FILTER_WIDTH,
    matrix object_itfm = matrix(0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0),
    output point Generated = point(0.0, 0.0, 0.0),
    output point UV = point(0.0, 0.0, 0.0),
    output point Object = point(0.0, 0.0, 0.0),
    output point Camera = point(0.0, 0.0, 0.0),
    output point Window = point(0.0, 0.0, 0.0),
    output normal Normal = normal(0.0, 0.0, 0.0),
    output point Reflection = point(0.0, 0.0, 0.0)
)
```

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| is_background | int | `0` | 是否为背景着色器，1 为是 |
| is_volume | int | `0` | 是否为体积着色器，1 为是 |
| from_dupli | int | `0` | 是否从复制体（实例化对象）获取坐标，1 为是 |
| use_transform | int | `0` | 是否使用自定义变换矩阵，1 为是 |
| bump_offset | string | `"center"` | 凹凸贴图偏移方向，可选 `"center"`、`"dx"`、`"dy"` |
| bump_filter_width | float | `BUMP_FILTER_WIDTH` | 凹凸贴图滤波宽度 |
| object_itfm | matrix | 零矩阵 | 自定义物体逆变换矩阵，当 `use_transform` 启用时使用 |

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| Generated | point | 生成坐标，基于物体包围盒自动生成的 [0,1] 范围坐标 |
| UV | point | UV 纹理坐标 |
| Object | point | 物体空间坐标 |
| Camera | point | 相机空间坐标 |
| Window | point | 窗口/屏幕空间坐标（NDC），Z 分量强制为 0 |
| Normal | normal | 法线方向，非背景模式下变换到物体空间并归一化 |
| Reflection | point | 反射方向向量 |

## 实现逻辑

### 背景模式（`is_background == 1`）

| 输出 | 计算方式 |
|------|----------|
| Generated | 直接使用 `P` |
| UV | 固定为 `(0, 0, 0)` |
| Object | 使用自定义矩阵变换 `P` 或直接使用 `P` |
| Camera | 将 `P` 加上相机世界位置后变换到相机空间 |
| Window | 从 `NDC` 属性获取 |
| Normal | 直接使用 `N` |
| Reflection | 直接使用入射方向 `I` |

### 非背景模式

1. **Generated 和 UV 的来源选择**：
   - **复制体模式**：从 `geom:dupli_generated` 和 `geom:dupli_uv` 获取。
   - **体积模式**：将 `P` 变换到物体空间作为 Generated，若存在生成变换矩阵则额外应用；UV 从 `geom:uv` 获取。
   - **普通模式**：尝试从 `geom:generated` 获取 Generated，失败则回退到物体空间变换；尝试从 `geom:uv` 获取 UV，若为灯光对象则使用重心坐标。

2. **Object**：使用自定义矩阵或 `transform("object", P)` 变换。

3. **Camera**：通过 `transform("camera", P)` 变换到相机空间。

4. **Window**：从 `NDC` 属性获取，最终 Z 分量被强制设为 0。

5. **Normal**：变换到物体空间并归一化：`normalize(transform("world", "object", N))`。

6. **Reflection**：计算入射光线关于法线的反射方向：`-reflect(I, N)`。

### 凹凸偏移处理

对 Generated、UV（非复制体）、Object、Camera、Window 和 Normal 施加 `Dx`/`Dy` 微分偏移。法线的凹凸偏移需要从 `geom:bump_map_normal` 属性获取凹凸法线。

## 对应 SVM 节点

对应 Cycles SVM（着色器虚拟机）中的 `NODE_TEX_COORD` 和 `NODE_TEX_COORD_BUMP_DX` / `NODE_TEX_COORD_BUMP_DY` 节点。
