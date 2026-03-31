# node_normal_map.osl - 法线贴图着色器

## 概述

该着色器将法线贴图纹理（通常为切线空间的蓝紫色图像）转换为实际的表面法线。支持多种空间模式：切线空间（Tangent）、物体空间（Object）、世界空间（World），以及 Blender 特有的物体空间和世界空间变体。强度参数控制法线贴图对最终法线的影响程度。

## 着色器签名

```osl
shader node_normal_map(float Strength = 1.0,
                       color Color = color(0.5, 0.5, 1.0),
                       string space = "tangent",
                       string attr_name = "geom:undisplaced_tangent",
                       string attr_sign_name = "geom:undisplaced_tangent_sign",
                       output normal Normal = N)
```

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| Strength | float | 1.0 | 法线贴图强度。0.0 = 使用原始法线，1.0 = 完全应用法线贴图 |
| Color | color | (0.5, 0.5, 1.0) | 法线贴图颜色值（默认值对应朝上的平面法线） |
| space | string | "tangent" | 法线空间类型：`"tangent"`、`"object"`、`"world"`、`"blender_object"`、`"blender_world"` |
| attr_name | string | "geom:undisplaced_tangent" | 切线属性名称 |
| attr_sign_name | string | "geom:undisplaced_tangent_sign" | 切线符号属性名称 |

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| Normal | normal | 计算得到的世界空间法线 |

## 实现逻辑

### 颜色解码
首先将法线贴图颜色从 [0, 1] 范围解码到 [-1, 1] 范围：`mcolor = 2.0 * (Color - 0.5)`。

### 切线空间模式（`"tangent"`）
这是最常用的模式，实现最为复杂：

1. 获取表面平滑状态 `is_smooth`。
2. 获取插值法线 `ninterp`：
   - 非平滑表面：使用几何法线 `Ng` 变换到物体空间。
   - 平滑表面：通过 `getattribute("geom:normal_map_normal")` 获取。
3. 获取切线向量 `tangent` 和切线符号 `tangent_sign`。
4. 计算副切线：`B = tangent_sign * cross(ninterp, tangent)`。
5. 应用强度：X 和 Y 分量乘以 `Strength`，Z 分量使用 `mix(1.0, Z, clamp(Strength, 0, 1))` 混合。
6. 构建最终法线：`Normal = normalize(mcolor[0] * tangent + mcolor[1] * B + mcolor[2] * ninterp)`。
7. 变换到世界空间：`Normal = normalize(transform("object", "world", Normal))`。

### 物体空间模式（`"object"`）
将解码后的颜色向量从物体空间变换到世界空间并归一化。

### 世界空间模式（`"world"`）
直接将解码后的颜色向量归一化作为世界空间法线。

### Blender 物体/世界空间模式
与标准模式类似，但翻转 Y 和 Z 分量（Blender 的特殊坐标约定）。

### 背面处理
- 对于背面多边形（`backfacing() == true`），最终法线取反。
- 在切线空间模式中，还需要在获取几何法线时考虑背面翻转。

### 强度混合（非切线空间）
当 `Strength != 1.0` 且空间不为切线空间时：`Normal = normalize(N + (Normal - N) * max(Strength, 0.0))`。

## 对应 SVM 节点

对应 Cycles SVM 中的 `NODE_NORMAL_MAP` 节点。该节点在 Blender 节点编辑器中显示为"法线贴图"（Normal Map）节点。
