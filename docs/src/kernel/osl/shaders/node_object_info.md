# node_object_info.osl - 物体信息着色器

## 概述

物体信息着色器（Object Info Node）用于获取当前被渲染物体的属性信息，包括位置、颜色、透明度、物体索引、材质索引以及随机值。该着色器是开放着色语言（OSL）中的输入类节点，常用于基于物体属性驱动材质变化的场景。

## 着色器签名

```osl
shader node_object_info(
    output point Location = point(0.0, 0.0, 0.0),
    output color Color = color(1.0, 1.0, 1.0),
    output float Alpha = 1.0,
    output float ObjectIndex = 0.0,
    output float MaterialIndex = 0.0,
    output float Random = 0.0
)
```

## 输入参数

无输入参数。

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| Location | point | 物体的世界空间原点位置 |
| Color | color | 物体的视口显示颜色（Blender 中物体属性面板设置的颜色） |
| Alpha | float | 物体的透明度值 |
| ObjectIndex | float | 物体的通道索引（Pass Index），可用于合成中的遮罩 |
| MaterialIndex | float | 材质的通道索引（Pass Index） |
| Random | float | 每个物体实例的随机值，取值范围 [0, 1] |

## 实现逻辑

所有输出参数均通过 `getattribute` 函数从渲染器内部获取：

- `"object:location"` → 物体位置
- `"object:color"` → 物体颜色
- `"object:alpha"` → 物体透明度
- `"object:index"` → 物体索引
- `"material:index"` → 材质索引
- `"object:random"` → 物体随机值

实现非常直接，无额外计算逻辑。所有属性均从 Cycles 渲染器的物体数据结构中读取。

## 对应 SVM 节点

对应 Cycles SVM（着色器虚拟机）中的 `NODE_OBJECT_INFO` 节点。
