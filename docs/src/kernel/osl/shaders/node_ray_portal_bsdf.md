# node_ray_portal_bsdf.osl - 光线传送门双向散射分布函数(BSDF)着色器

## 概述

该着色器实现了光线传送门效果，将到达表面的光线传送到指定的空间位置并沿指定方向继续传播。可用于实现虚拟窗口、传送门、光路重定向等特殊效果。

## 着色器签名

```osl
shader node_ray_portal_bsdf(color Color = 0.8,
                            vector Position = vector(0.0, 0.0, 0.0),
                            vector Direction = vector(0.0, 0.0, 0.0),
                            output closure color BSDF = 0)
```

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| Color | color | 0.8 | 传送门滤光颜色 |
| Position | vector | (0.0, 0.0, 0.0) | 光线传送目标位置 |
| Direction | vector | (0.0, 0.0, 0.0) | 光线传送后的方向 |

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| BSDF | closure color | 光线传送门闭包(Closure)输出 |

## 实现逻辑

直接使用 `ray_portal_bsdf(Position, Direction)` 闭包，乘以颜色权重。该闭包将光线的起点重设为 Position，方向重设为 Direction，实现空间传送效果。

## 对应 SVM 节点

对应 SVM 中的 `NODE_CLOSURE_BSDF`，闭包类型为 `CLOSURE_BSDF_RAY_PORTAL_ID`。
