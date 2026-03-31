# node_vector_rotate.osl - 向量旋转着色器

## 概述

该着色器围绕指定的旋转中心对输入向量进行旋转操作。支持多种旋转模式：绕任意轴旋转、绕固定坐标轴（X/Y/Z）旋转以及通过欧拉角旋转。同时支持反向旋转。依赖 `node_math.h` 头文件中定义的 `euler_to_mat()` 等辅助函数。

## 着色器签名

```osl
shader node_vector_rotate(int invert = 0,
                          string rotate_type = "axis",
                          vector VectorIn = vector(0.0, 0.0, 0.0),
                          point Center = point(0.0, 0.0, 0.0),
                          point Rotation = point(0.0, 0.0, 0.0),
                          vector Axis = vector(0.0, 0.0, 1.0),
                          float Angle = 0.0,
                          output vector VectorOut = vector(0.0, 0.0, 0.0))
```

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| invert | int | 0 | 是否反转旋转方向。0 = 正向旋转，非零 = 反向旋转 |
| rotate_type | string | "axis" | 旋转模式：`"axis"`（任意轴）、`"x_axis"`、`"y_axis"`、`"z_axis"`、`"euler_xyz"` |
| VectorIn | vector | (0, 0, 0) | 待旋转的输入向量 |
| Center | point | (0, 0, 0) | 旋转中心点 |
| Rotation | point | (0, 0, 0) | 欧拉角旋转值（仅 `"euler_xyz"` 模式使用，弧度制） |
| Axis | vector | (0, 0, 1) | 自定义旋转轴（仅 `"axis"` 模式使用） |
| Angle | float | 0.0 | 旋转角度（弧度制，除 `"euler_xyz"` 模式外使用） |

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| VectorOut | vector | 旋转后的输出向量 |

## 实现逻辑

所有旋转操作都以 `Center` 为中心：先将向量平移到以中心为原点的坐标系，执行旋转，再平移回去。

### 欧拉角模式（`"euler_xyz"`）
1. 调用 `euler_to_mat(Rotation)` 将欧拉角转换为旋转矩阵。
2. 如果 `invert` 非零，对旋转矩阵取转置（等价于求逆，因为旋转矩阵是正交矩阵）。
3. 执行变换：`VectorOut = transform(rmat, VectorIn - Center) + Center`。

### 固定轴和自定义轴模式
1. 如果 `invert` 非零，将角度取反：`a = -Angle`。
2. 根据 `rotate_type` 选择旋转轴：
   - `"x_axis"`：轴为 (1, 0, 0)
   - `"y_axis"`：轴为 (0, 1, 0)
   - `"z_axis"`：轴为 (0, 0, 1)
   - `"axis"`：使用用户指定的 `Axis`（当 `Axis` 长度为零时返回原向量）
3. 使用开放着色语言（OSL）内置的 `rotate()` 函数执行绕轴旋转。

## 对应 SVM 节点

对应 Cycles SVM 中的 `NODE_VECTOR_ROTATE` 节点。该节点在 Blender 节点编辑器中显示为"向量旋转"（Vector Rotate）节点。
