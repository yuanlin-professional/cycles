# node_point_info.osl - 点云信息着色器

## 概述

点云信息着色器（Point Info Node）用于获取点云（Point Cloud）几何体中各点的属性信息。该着色器是开放着色语言（OSL）中的输入类节点，仅在渲染点云对象时有效，可获取每个点的位置、半径和随机值。

## 着色器签名

```osl
shader node_point_info(
    output point Position = point(0.0, 0.0, 0.0),
    output float Radius = 0.0,
    output float Random = 0.0
)
```

## 输入参数

无输入参数。

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| Position | point | 点云中当前点的世界空间位置 |
| Radius | float | 当前点的半径大小 |
| Random | float | 每个点的随机值，取值范围 [0, 1] |

## 实现逻辑

所有输出参数均通过 `getattribute` 函数从渲染器的几何体数据中获取：

- `"geom:point_position"` → 点位置
- `"geom:point_radius"` → 点半径
- `"geom:point_random"` → 点随机值

实现非常直接，无额外计算逻辑。

## 对应 SVM 节点

对应 Cycles SVM（着色器虚拟机）中的 `NODE_POINT_INFO` 节点。
