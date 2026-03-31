# node_wireframe.osl - 线框着色器

## 概述

线框着色器（Wireframe Node）用于生成网格的线框效果。该着色器是开放着色语言（OSL）中的输入类节点，基于三角形边缘检测输出线框遮罩值，可用于渲染线框叠加效果或作为程序化纹理使用。

## 着色器签名

```osl
shader node_wireframe(
    string bump_offset = "center",
    float bump_filter_width = BUMP_FILTER_WIDTH,
    int use_pixel_size = 0,
    float Size = 0.01,
    output float Fac = 0.0
)
```

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| bump_offset | string | `"center"` | 凹凸贴图偏移方向，可选 `"center"`、`"dx"`、`"dy"` |
| bump_filter_width | float | `BUMP_FILTER_WIDTH` | 凹凸贴图滤波宽度 |
| use_pixel_size | int | `0` | 是否使用像素单位，1 表示线宽以像素为单位，0 表示以世界空间单位 |
| Size | float | `0.01` | 线框宽度 |

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| Fac | float | 线框遮罩值，在三角形边缘处接近 1.0，内部为 0.0 |

## 实现逻辑

1. **凹凸偏移**：在计算线框之前，若 `bump_offset` 为 `"dx"` 或 `"dy"`，先对着色点位置 `P` 施加微分偏移。

2. **线框计算**：调用开放着色语言（OSL）内建函数 `wireframe("triangles", Size, use_pixel_size)` 计算线框因子。该函数基于着色点到最近三角形边缘的距离和线宽参数返回遮罩值。

## 对应 SVM 节点

对应 Cycles SVM（着色器虚拟机）中的 `NODE_WIREFRAME` 节点。
