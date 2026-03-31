# node_map_range.osl - 映射范围着色器

## 概述

该着色器将输入值从一个范围重新映射到另一个范围。支持四种插值模式：线性插值（Linear）、阶梯插值（Stepped）、平滑阶梯插值（Smoothstep）和更平滑阶梯插值（Smootherstep）。着色器内部定义了安全除法和更平滑阶梯函数的辅助实现。

## 着色器签名

```osl
shader node_map_range(string range_type = "linear",
                      float Value = 1.0,
                      float FromMin = 0.0,
                      float FromMax = 1.0,
                      float ToMin = 0.0,
                      float ToMax = 1.0,
                      float Steps = 4.0,
                      output float Result = 0.0)
```

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| range_type | string | "linear" | 映射类型：`"linear"`、`"stepped"`、`"smoothstep"`、`"smootherstep"` |
| Value | float | 1.0 | 待映射的输入值 |
| FromMin | float | 0.0 | 源范围最小值 |
| FromMax | float | 1.0 | 源范围最大值 |
| ToMin | float | 0.0 | 目标范围最小值 |
| ToMax | float | 1.0 | 目标范围最大值 |
| Steps | float | 4.0 | 阶梯数量（仅 `"stepped"` 模式使用） |

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| Result | float | 映射后的输出值 |

## 实现逻辑

当 `FromMax != FromMin` 时（避免除以零），根据不同模式计算映射因子 `Factor`：

1. **线性模式**（`"linear"`）：
   - `Factor = (Value - FromMin) / (FromMax - FromMin)`
   - 简单的线性归一化。

2. **阶梯模式**（`"stepped"`）：
   - 先线性归一化，然后量化到离散阶梯。
   - `Factor = floor(Factor * (Steps + 1)) / Steps`
   - `Steps <= 0` 时结果为 0。

3. **平滑阶梯模式**（`"smoothstep"`）：
   - 使用 Hermite 插值函数 `smoothstep()`。
   - 当 `FromMin > FromMax` 时自动反转方向。

4. **更平滑阶梯模式**（`"smootherstep"`）：
   - 使用 Ken Perlin 的改进版平滑函数：`t^3 * (t * (t * 6 - 15) + 10)`。
   - 具有零一阶和二阶导数的边界条件，过渡更加平滑。

最终结果：`Result = ToMin + Factor * (ToMax - ToMin)`。

### 辅助函数

文件内部定义了两个辅助函数：
- `safe_divide(a, b)`：安全除法，`b == 0` 时返回 0。
- `smootherstep(edge0, edge1, x)`：更平滑的阶梯函数实现。

## 对应 SVM 节点

对应 Cycles SVM 中的 `NODE_MAP_RANGE` 节点。该节点在 Blender 节点编辑器中显示为"映射范围"（Map Range）节点。
