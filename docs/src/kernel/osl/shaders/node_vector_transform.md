# node_vector_transform.osl - 向量变换着色器

## 概述

该着色器在不同坐标空间之间变换向量、法线或点。支持世界空间（World）、物体空间（Object）和相机空间（Camera）之间的相互转换。根据变换类型的不同，对向量和法线分别进行适当的处理。

## 着色器签名

```osl
shader node_vector_transform(string transform_type = "vector",
                             string convert_from = "world",
                             string convert_to = "object",
                             vector VectorIn = vector(0.0, 0.0, 0.0),
                             output vector VectorOut = vector(0.0, 0.0, 0.0))
```

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| transform_type | string | "vector" | 变换类型：`"vector"`（向量）、`"point"`（点）、`"normal"`（法线） |
| convert_from | string | "world" | 源坐标空间：`"world"`、`"object"`、`"camera"` |
| convert_to | string | "object" | 目标坐标空间：`"world"`、`"object"`、`"camera"` |
| VectorIn | vector | (0, 0, 0) | 待变换的输入向量 |

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| VectorOut | vector | 变换后的输出向量 |

## 实现逻辑

根据 `transform_type` 采用不同的变换方式：

1. **向量模式**（`"vector"`）：
   - 使用 `transform(convert_from, convert_to, VectorIn)` 进行坐标空间变换。
   - 向量变换只考虑旋转和缩放，不受平移影响。

2. **法线模式**（`"normal"`）：
   - 先执行与向量相同的 `transform()` 变换。
   - 然后调用 `normalize()` 归一化结果，确保法线为单位向量。
   - 法线在非均匀缩放下需要特殊处理以保持垂直性。

3. **点模式**（`"point"`）：
   - 将输入向量强制转换为 `point` 类型。
   - 使用 `transform(convert_from, convert_to, Point)` 变换。
   - 点变换包含平移分量，与向量变换不同。

## 对应 SVM 节点

对应 Cycles SVM 中的 `NODE_VECTOR_TRANSFORM` 节点。该节点在 Blender 节点编辑器中显示为"向量变换"（Vector Transform）节点。
