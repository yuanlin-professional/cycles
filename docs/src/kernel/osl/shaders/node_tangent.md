# node_tangent.osl - 切线着色器

## 概述

切线着色器（Tangent Node）用于生成着色点的切线向量。该着色器是开放着色语言（OSL）中的输入类节点，支持从 UV 贴图或径向方向两种方式生成切线，常用于各向异性（Anisotropic）着色。

## 着色器签名

```osl
shader node_tangent(
    string attr_name = "geom:tangent",
    string direction_type = "radial",
    string axis = "z",
    output normal Tangent = normalize(dPdu)
)
```

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| attr_name | string | `"geom:tangent"` | UV 贴图模式下使用的切线属性名称 |
| direction_type | string | `"radial"` | 切线方向类型，可选 `"radial"`（径向）或 `"uv_map"`（UV 贴图） |
| axis | string | `"z"` | 径向模式下的参考轴，可选 `"x"`、`"y"`、`"z"` |

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| Tangent | normal | 计算得到的切线方向向量（世界空间，已归一化） |

## 实现逻辑

1. **UV 贴图模式**（`direction_type == "uv_map"`）：
   - 通过 `getattribute(attr_name, T)` 直接从指定属性读取切线向量。

2. **径向模式**（`direction_type == "radial"`）：
   - 首先尝试获取生成坐标 `geom:generated`，若不存在则使用着色点位置 `P`。
   - 根据 `axis` 参数选择不同的轴向计算方式：
     - `"x"` 轴：`T = (0, -(z-0.5), (y-0.5))`
     - `"y"` 轴：`T = (-(z-0.5), 0, (x-0.5))`
     - `"z"` 轴：`T = (-(y-0.5), (x-0.5), 0)`

3. **空间变换与正交化**：
   - 将切线从物体空间变换到世界空间：`transform("object", "world", T)`。
   - 通过双重叉乘使切线与法线 `N` 正交：`cross(N, normalize(cross(T, N)))`。

## 对应 SVM 节点

对应 Cycles SVM（着色器虚拟机）中的 `NODE_TANGENT` 节点。
