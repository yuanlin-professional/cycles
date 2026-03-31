# node_vector_displacement.osl - 向量置换着色器

## 概述

该着色器根据输入的向量贴图计算表面置换偏移。与标量置换不同，向量置换可以在三维空间的任意方向偏移表面顶点，支持切线空间（Tangent）、物体空间（Object）和世界空间（World）三种模式。

## 着色器签名

```osl
shader node_vector_displacement(color Vector = color(0.0, 0.0, 0.0),
                                float Midlevel = 0.0,
                                float Scale = 1.0,
                                string space = "tangent",
                                string attr_name = "geom:tangent",
                                string attr_sign_name = "geom:tangent_sign",
                                output vector Displacement = vector(0.0, 0.0, 0.0))
```

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| Vector | color | (0, 0, 0) | 向量置换贴图颜色值 |
| Midlevel | float | 0.0 | 中间级别，表示无置换时的基准值 |
| Scale | float | 1.0 | 置换缩放因子 |
| space | string | "tangent" | 置换空间：`"tangent"`、`"object"`、`"world"` |
| attr_name | string | "geom:tangent" | 切线属性名称 |
| attr_sign_name | string | "geom:tangent_sign" | 切线符号属性名称 |

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| Displacement | vector | 世界空间中的置换偏移向量 |

## 实现逻辑

1. 计算基础偏移：`offset = (Vector - Midlevel) * Scale`。

### 切线空间模式（`"tangent"`）
1. 将着色法线 `N` 从世界空间变换到物体空间并归一化：`N_object`。
2. 获取切线向量 `T_object`：
   - 优先通过 `getattribute(attr_name)` 获取预计算切线。
   - 若不可用，使用 UV 导数 `dPdu` 的归一化值。
3. 计算副切线：`B_object = normalize(cross(N_object, T_object))`。
4. 如有切线符号属性，将副切线乘以符号值以处理镜像 UV。
5. 在 TBN 基中重建置换：`Displacement = T * offset[0] + N * offset[1] + B * offset[2]`。

### 物体空间和世界空间模式
- 物体空间和世界空间模式直接使用 `offset` 作为置换向量。

### 空间转换
- 非世界空间模式（切线空间和物体空间）最终都将置换向量从物体空间变换到世界空间：`Displacement = transform("object", "world", Displacement)`。

## 对应 SVM 节点

对应 Cycles SVM 中的 `NODE_VECTOR_DISPLACEMENT` 节点。该节点在 Blender 节点编辑器中显示为"向量置换"（Vector Displacement）节点，通常连接到材质输出的置换接口。
