# node_mix.osl - 颜色混合着色器（旧版）

## 概述

该着色器实现了 18 种颜色混合模式（旧版混合节点），将两个输入颜色按指定的混合类型和混合因子进行混合。这是 Cycles 渲染器中功能最丰富的颜色混合工具，涵盖了 Photoshop 等图像处理软件中常见的所有混合模式。属于开放着色语言（OSL）颜色工具节点。

## 着色器签名

```osl
shader node_mix(
    string mix_type = "mix",
    int use_clamp = 0,
    float Fac = 0.5,
    color Color1 = 0.0,
    color Color2 = 0.0,
    output color Color = 0.0)
```

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| mix_type | string | "mix" | 混合模式类型（见下方支持的混合模式列表） |
| use_clamp | int | 0 | 是否将输出结果钳制到 [0, 1] 范围 |
| Fac | float | 0.5 | 混合因子，自动钳制到 [0, 1] 范围 |
| Color1 | color | 0.0 | 第一个输入颜色（基础色） |
| Color2 | color | 0.0 | 第二个输入颜色（混合色） |

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| Color | color | 混合后的输出颜色 |

## 支持的混合模式

| 模式名 | 中文名称 | 说明 |
|--------|----------|------|
| mix | 混合 | 线性插值混合 |
| add | 相加 | 将两个颜色相加 |
| multiply | 相乘 | 将两个颜色逐通道相乘 |
| screen | 滤色 | 反转相乘后再反转 |
| overlay | 叠加 | 根据基础色明暗选择相乘或滤色 |
| subtract | 相减 | 从基础色中减去混合色 |
| divide | 相除 | 基础色逐通道除以混合色 |
| difference | 差值 | 两颜色之差的绝对值 |
| exclusion | 排除 | 类似差值但对比度更低 |
| darken | 变暗 | 逐通道取较小值 |
| lighten | 变亮 | 逐通道取较大值 |
| dodge | 颜色减淡 | 模拟摄影减淡效果 |
| burn | 颜色加深 | 模拟摄影加深效果 |
| hue | 色相 | 使用混合色的色相与基础色的饱和度/明度 |
| saturation | 饱和度 | 使用混合色的饱和度与基础色的色相/明度 |
| value | 明度 | 使用混合色的明度与基础色的色相/饱和度 |
| color | 颜色 | 使用混合色的色相和饱和度与基础色的明度 |
| soft_light | 柔光 | 柔和的高光/阴影效果 |
| linear_light | 线性光 | 根据混合色明暗进行线性加减 |

## 辅助函数

文件中定义了 19 个内部混合函数和 1 个钳制函数：

- `node_mix_blend` - 线性插值
- `node_mix_add` / `node_mix_sub` - 加/减法
- `node_mix_mul` / `node_mix_div` - 乘/除法
- `node_mix_screen` - 滤色
- `node_mix_overlay` - 叠加
- `node_mix_diff` / `node_mix_exclusion` - 差值/排除
- `node_mix_dark` / `node_mix_light` - 变暗/变亮
- `node_mix_dodge` / `node_mix_burn` - 减淡/加深
- `node_mix_hue` / `node_mix_sat` / `node_mix_val` / `node_mix_color` - HSV 分量混合
- `node_mix_soft` / `node_mix_linear` - 柔光/线性光
- `node_mix_clamp` - 结果钳制

## 实现逻辑

1. 将混合因子 `Fac` 钳制到 [0, 1] 范围。
2. 根据 `mix_type` 字符串选择对应的混合函数执行计算。
3. 若 `use_clamp` 启用，将结果的每个通道钳制到 [0, 1]。

## 对应 SVM 节点

对应 SVM 内核中的 `NODE_MIX` 节点，类型为 `SHADER_NODE_MIX_RGB`（旧版）。
