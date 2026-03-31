# node_normal.osl - 法线着色器

## 概述

该着色器输出归一化的方向向量，并计算该方向与输入法线之间的点积。常用于基于法线方向创建简单的光照或遮罩效果。

## 着色器签名

```osl
shader node_normal(normal direction = normal(0.0, 0.0, 0.0),
                   normal NormalIn = normal(0.0, 0.0, 0.0),
                   output normal NormalOut = normal(0.0, 0.0, 0.0),
                   output float Dot = 1.0)
```

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| direction | normal | (0, 0, 0) | 预设的方向向量（由节点内部 UI 控制） |
| NormalIn | normal | (0, 0, 0) | 输入法线，通常连接到几何体的法线数据 |

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| NormalOut | normal | 归一化后的方向向量 |
| Dot | float | 方向向量与输入法线的点积值，范围 [-1, 1] |

## 实现逻辑

1. 归一化方向向量：`NormalOut = normalize(direction)`。
2. 计算点积：`Dot = dot(NormalOut, normalize(NormalIn))`。
   - 点积值为 1.0 时表示两个法线同向。
   - 点积值为 0.0 时表示两个法线垂直。
   - 点积值为 -1.0 时表示两个法线反向。

该节点输出的点积可用于驱动混合系数或遮罩，实现基于方向的效果（如顶部积雪、侧面风化等）。

## 对应 SVM 节点

对应 Cycles SVM 中的 `NODE_NORMAL` 节点。该节点在 Blender 节点编辑器中显示为"法向"（Normal）节点。
