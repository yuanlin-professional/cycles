# node_bevel.osl - 倒角着色器

## 概述

倒角着色器（Bevel Node）用于在着色阶段模拟边缘倒角效果，无需修改实际几何体。该着色器是开放着色语言（OSL）中的输入类节点，通过采样邻近表面的法线来生成平滑的倒角法线，常用于在不增加几何复杂度的情况下柔化硬边缘。

## 着色器签名

```osl
shader node_bevel(
    int samples = 4,
    float Radius = 0.05,
    normal NormalIn = N,
    output normal NormalOut = N
)
```

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| samples | int | `4` | 采样数量，值越大倒角效果越平滑但计算越慢 |
| Radius | float | `0.05` | 倒角半径，控制边缘柔化的范围 |
| NormalIn | normal | `N` | 输入法线方向，默认为着色点法线 |

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| NormalOut | normal | 经过倒角处理后的法线方向 |

## 实现逻辑

1. **倒角法线采样**：通过特殊的 `texture("@bevel", samples, Radius)` 调用获取倒角法线向量。返回值经过 `color` 到 `normal` 的类型转换。这是一种利用纹理调用接口传递自定义参数的技巧（代码注释中标注为 "Abuse texture call"）。

2. **法线合成**：将采样得到的倒角法线与输入法线合成：
   - 计算倒角偏移量：`bevel_N - N`（倒角法线与原始几何法线的差值）。
   - 将偏移量叠加到输入法线：`NormalIn + (bevel_N - N)`。
   - 归一化：`normalize(...)`。

   这种方式可以保留输入法线的自定义修改（如法线贴图），同时叠加倒角效果。

## 对应 SVM 节点

对应 Cycles SVM（着色器虚拟机）中的 `NODE_BEVEL` 节点。
