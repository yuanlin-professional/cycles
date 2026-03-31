# node_radial_tiling.osl - 径向平铺着色器

## 概述

该开放着色语言(OSL)着色器节点实现径向平铺功能。将二维平面坐标划分为围绕原点均匀分布的扇形段落，输出每个段落的局部坐标、段落 ID、段落宽度和旋转角度。支持可变边数和圆角度，可用于创建万花筒效果、径向对称图案、正多边形纹理等。

## 着色器签名

```osl
shader node_radial_tiling(
    int use_normalize = 0,
    point Vector = P,
    float Sides = 5.0,
    float Roundness = 0.0,
    output point SegmentCoordinates = point(0.0, 0.0, 0.0),
    output float SegmentID = 0.0,
    output float SegmentWidth = 0.0,
    output float SegmentRotation = 0.0)
```

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| use_normalize | int | 0 | 是否归一化段落坐标 |
| Vector | point | P | 输入纹理坐标（使用 X 和 Y 分量） |
| Sides | float | 5.0 | 径向分段数（最小值 2.0），即将 360 度均分为多少段 |
| Roundness | float | 0.0 | 圆角度（0.0 = 多边形边缘，1.0 = 完全圆形），范围 [0, 1] |

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| SegmentCoordinates | point | 当前段落的局部坐标（XY 平面，Z = 0） |
| SegmentID | float | 当前段落的 ID 编号 |
| SegmentWidth | float | 当前段落的宽度 |
| SegmentRotation | float | 当前段落相对于 X 轴的旋转角度 |

## 实现逻辑

1. **连接检测优化**：通过 `isconnected()` 检测哪些输出端口已连接，仅计算有连接的输出，避免不必要的计算开销。
2. **段落坐标计算**：调用 `calculate_out_variables` 函数（定义于 `node_radial_tiling_shared.h`），基于输入的 2D 坐标、边数和圆角度，计算：
   - 段落内局部坐标（r-gon 参数域）
   - 段落宽度（最大单位参数）
   - 段落旋转角度（X 轴到角平分线的角度）
3. **段落 ID 计算**：调用 `calculate_out_segment_id`，基于二维坐标和边数确定当前所在扇形段落的编号。
4. **参数限制**：Sides 最小值限制为 2.0，Roundness 限制在 [0, 1] 范围。

## 对应 SVM 节点

对应 `svm_radial_tiling` 相关实现（`src/kernel/svm/radial_tiling.h`）。
