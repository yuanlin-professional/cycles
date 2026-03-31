# node_vector_curves.osl - 向量曲线着色器

## 概述

该着色器通过预定义的颜色渐变（Ramp）曲线对输入向量的各分量进行映射变换。每个分量独立查找曲线表，实现非线性的向量值调整。支持外推功能和混合因子控制。依赖 `node_ramp_util.h` 头文件中的 `rgb_ramp_lookup()` 函数。

## 着色器签名

```osl
shader node_vector_curves(color ramp[] = {0.0},
                          float min_x = 0.0,
                          float max_x = 1.0,
                          int extrapolate = 1,
                          vector VectorIn = vector(0.0, 0.0, 0.0),
                          float Fac = 0.0,
                          output vector VectorOut = vector(0.0, 0.0, 0.0))
```

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| ramp | color[] | {0.0} | 曲线查找表数组，预计算的颜色渐变数据 |
| min_x | float | 0.0 | 曲线输入范围最小值 |
| max_x | float | 1.0 | 曲线输入范围最大值 |
| extrapolate | int | 1 | 是否启用外推。1 = 允许超出 [0, 1] 范围的外推 |
| VectorIn | vector | (0, 0, 0) | 输入向量 |
| Fac | float | 0.0 | 混合因子。0.0 = 输出原始向量，1.0 = 完全应用曲线 |

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| VectorOut | vector | 经过曲线映射并混合后的输出向量 |

## 实现逻辑

1. **归一化输入**：将输入向量各分量从 `[min_x, max_x]` 归一化到 `[0, 1]`：
   - `c = (VectorIn - min_x) / (max_x - min_x)`

2. **曲线查找**：对每个分量使用 `rgb_ramp_lookup()` 查找曲线表：
   - `r = rgb_ramp_lookup(ramp, c[0], 1, extrapolate)` -> 取 R 通道
   - `g = rgb_ramp_lookup(ramp, c[0], 1, extrapolate)` -> 取 G 通道
   - `b = rgb_ramp_lookup(ramp, c[0], 1, extrapolate)` -> 取 B 通道

3. **提取对应通道**：
   - `VectorOut[0] = r[0]`（查找结果的 R 通道）
   - `VectorOut[1] = g[1]`（查找结果的 G 通道）
   - `VectorOut[2] = b[2]`（查找结果的 B 通道）

4. **混合**：`VectorOut = mix(VectorIn, VectorOut, Fac)`。

## 对应 SVM 节点

对应 Cycles SVM 中的 `NODE_VECTOR_CURVES` 节点。该节点在 Blender 节点编辑器中显示为"向量曲线"（Vector Curves）节点。
