# node_hair_info.osl - 毛发信息着色器

## 概述

毛发信息着色器（Hair Info Node）用于获取毛发/曲线几何体的属性信息。该着色器是开放着色语言（OSL）中的输入类节点，仅在渲染毛发或曲线对象时有效，可获取曲线的截距位置、长度、粗细、切线法线和随机值等信息。

## 着色器签名

```osl
shader node_hair_info(
    output float IsStrand = 0.0,
    output float Intercept = 0.0,
    output float Length = 0.0,
    output float Thickness = 0.0,
    output normal TangentNormal = N,
    output float Random = 0
)
```

## 输入参数

无输入参数。

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| IsStrand | float | 是否为毛发/曲线，1.0 表示当前几何体是曲线 |
| Intercept | float | 沿曲线的截距位置，取值范围 [0, 1]，0 为根部，1 为末梢 |
| Length | float | 曲线的总长度 |
| Thickness | float | 曲线在当前着色点处的粗细 |
| TangentNormal | normal | 曲线的切线法线方向，默认值为着色点法线 `N` |
| Random | float | 每根曲线的随机值，取值范围 [0, 1] |

## 实现逻辑

所有输出参数均通过 `getattribute` 函数从渲染器的曲线几何体数据中获取：

- `"geom:is_curve"` → 是否为曲线
- `"geom:curve_intercept"` → 曲线截距
- `"geom:curve_length"` → 曲线长度
- `"geom:curve_thickness"` → 曲线粗细
- `"geom:curve_tangent_normal"` → 曲线切线法线
- `"geom:curve_random"` → 曲线随机值

实现非常直接，无额外计算逻辑。

## 对应 SVM 节点

对应 Cycles SVM（着色器虚拟机）中的 `NODE_HAIR_INFO` 节点。
