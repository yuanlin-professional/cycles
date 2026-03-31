# node_vector_map_range.osl - 向量映射范围着色器

## 概述

该着色器将输入向量的每个分量分别从一个范围重新映射到另一个范围。支持四种插值模式：线性（Linear）、阶梯（Stepped）、平滑阶梯（Smoothstep）和更平滑阶梯（Smootherstep）。每个分量独立计算，允许各轴使用不同的源和目标范围。可选的钳制功能确保输出不超出目标范围。

## 着色器签名

```osl
shader node_vector_map_range(string range_type = "linear",
                             int use_clamp = 0,
                             point VectorIn = point(1.0, 1.0, 1.0),
                             point From_Min_FLOAT3 = point(0.0, 0.0, 0.0),
                             point From_Max_FLOAT3 = point(1.0, 1.0, 1.0),
                             point To_Min_FLOAT3 = point(0.0, 0.0, 0.0),
                             point To_Max_FLOAT3 = point(1.0, 1.0, 1.0),
                             point Steps_FLOAT3 = point(4.0, 4.0, 4.0),
                             output point VectorOut = point(0.0, 0.0, 0.0))
```

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| range_type | string | "linear" | 映射类型：`"linear"`、`"stepped"`、`"smoothstep"`、`"smootherstep"` |
| use_clamp | int | 0 | 是否启用输出钳制。非零值启用钳制 |
| VectorIn | point | (1, 1, 1) | 待映射的输入向量 |
| From_Min_FLOAT3 | point | (0, 0, 0) | 源范围最小值（每个分量独立） |
| From_Max_FLOAT3 | point | (1, 1, 1) | 源范围最大值（每个分量独立） |
| To_Min_FLOAT3 | point | (0, 0, 0) | 目标范围最小值（每个分量独立） |
| To_Max_FLOAT3 | point | (1, 1, 1) | 目标范围最大值（每个分量独立） |
| Steps_FLOAT3 | point | (4, 4, 4) | 各分量的阶梯数量（仅 `"stepped"` 模式使用） |

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| VectorOut | point | 映射后的输出向量 |

## 实现逻辑

对向量的 X、Y、Z 三个分量分别执行范围映射：

1. **线性模式**（`"linear"`）：
   - `factor = (VectorIn - from_min) / (from_max - from_min)`（安全除法）

2. **阶梯模式**（`"stepped"`）：
   - 先线性归一化，然后按每个分量的阶梯数量化。
   - 各分量独立计算：`floor(factor[i] * (steps[i] + 1)) / steps[i]`

3. **平滑阶梯模式**（`"smoothstep"`）：
   - 线性归一化后钳制到 [0, 1]，然后应用 Hermite 插值：`(3 - 2t) * t^2`。

4. **更平滑阶梯模式**（`"smootherstep"`）：
   - 线性归一化后钳制到 [0, 1]，然后应用 Perlin 改进插值：`t^3 * (t * (t * 6 - 15) + 10)`。

最终结果：`VectorOut = to_min + factor * (to_max - to_min)`。

**钳制处理**（`use_clamp > 0`）：对每个分量，根据 `to_min` 和 `to_max` 的大小关系选择正确的钳制方向，确保当目标范围反转（`to_min > to_max`）时仍能正确钳制。

### 辅助函数

文件内部定义了两个重载的安全除法函数：
- `safe_divide(float, float)`：标量安全除法。
- `safe_divide(point, point)`：逐分量安全除法。

## 对应 SVM 节点

对应 Cycles SVM 中的 `NODE_MAP_RANGE` 节点（向量版本）。该节点在 Blender 节点编辑器中显示为"映射范围"（Map Range）节点的向量模式。
