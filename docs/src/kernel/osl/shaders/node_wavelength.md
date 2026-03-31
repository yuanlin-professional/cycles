# node_wavelength.osl - 波长转颜色着色器

## 概述

该着色器将可见光波长（纳米）转换为对应的 RGB 颜色值。基于物理光谱的波长到颜色映射关系，用于创建物理上准确的光谱颜色。属于 Cycles 渲染器开放着色语言（OSL）转换工具节点。

## 着色器签名

```osl
shader node_wavelength(
    float Wavelength = 500.0,
    output color Color = 0.0)
```

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| Wavelength | float | 500.0 | 光的波长，单位为纳米（nm）。可见光范围约为 380nm（紫色）到 780nm（红色） |

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| Color | color | 对应波长的 RGB 颜色值 |

## 实现逻辑

直接调用开放着色语言（OSL）内置的 `wavelength_color()` 函数，该函数使用 CIE 1931 色彩匹配函数将单一波长映射为 RGB 颜色。

## 对应 SVM 节点

对应 SVM 内核中的 `NODE_WAVELENGTH` 节点，类型为 `SHADER_NODE_WAVELENGTH`。
