# map_range.h - 范围映射节点

## 概述

`map_range.h` 实现了 Cycles SVM 中的范围映射（Map Range）节点，对应 Blender 着色器编辑器中的"映射范围（Map Range）"节点。该节点将输入值从一个范围重新映射到另一个范围，支持线性、阶梯、平滑阶梯和更平滑阶梯四种插值模式。同时提供标量和向量两个版本。

## 核心函数

### `smootherstep`

```c
ccl_device_inline float smootherstep(const float edge0, const float edge1, float x)
```

- **功能**: Ken Perlin 的更平滑阶梯函数（5 次多项式）。
- **公式**: `x^3 * (x * (x * 6 - 15) + 10)`，其中 `x` 先被归一化到 [0, 1]。
- **特性**: 一阶和二阶导数在边界处均为零，比标准 `smoothstep` 更加平滑。

### `svm_node_map_range`

```c
ccl_device_noinline int svm_node_map_range(KernelGlobals kg,
                                           ccl_private float *stack,
                                           const uint value_stack_offset,
                                           const uint parameters_stack_offsets,
                                           const uint results_stack_offsets,
                                           int offset)
```

- **功能**: 标量范围映射节点的 SVM 执行入口。
- **输入参数**: 值(value)、源最小值(from_min)、源最大值(from_max)、目标最小值(to_min)、目标最大值(to_max)、步数(steps)。
- **插值模式**:
  - `NODE_MAP_RANGE_LINEAR`: 线性映射 `factor = (value - from_min) / (from_max - from_min)`
  - `NODE_MAP_RANGE_STEPPED`: 阶梯映射，将连续值量化为离散步级
  - `NODE_MAP_RANGE_SMOOTHSTEP`: Hermite 平滑插值（3 次多项式）
  - `NODE_MAP_RANGE_SMOOTHERSTEP`: Perlin 更平滑插值（5 次多项式）
- **输出**: `result = to_min + factor * (to_max - to_min)`
- **边界处理**: 当 `from_min == from_max` 时返回 0，避免除零。

### `svm_node_vector_map_range`

```c
ccl_device_noinline int svm_node_vector_map_range(ccl_private float *stack,
                                                  const uint value_stack_offset,
                                                  const uint parameters_stack_offsets,
                                                  const uint results_stack_offsets,
                                                  const int offset)
```

- **功能**: 向量范围映射节点的 SVM 执行入口，对三分量独立进行范围映射。
- **额外功能**: 支持可选的结果钳位（对于 SMOOTHSTEP 和 SMOOTHERSTEP 模式不需要钳位，因为其输出天然在 [0,1] 范围内）。钳位时考虑 `to_min > to_max` 的情况，自动交换边界。

## 依赖关系

- **内部头文件**:
  - `kernel/svm/util.h` — 栈操作、节点解包和默认值加载工具
- **被引用**:
  - `kernel/svm/svm.h` — SVM 主调度器，在 `NODE_MAP_RANGE` 和 `NODE_VECTOR_MAP_RANGE` 指令分支中调用

## 实现细节 / 关键算法

### 四种插值模式对比

| 模式 | 连续性 | 公式 | 特点 |
|------|--------|------|------|
| Linear | C0 | `(v - min) / (max - min)` | 线性，可能在边界处不平滑 |
| Stepped | 不连续 | `floor(factor * (steps+1)) / steps` | 量化为离散步级 |
| Smoothstep | C1 | `3x^2 - 2x^3` | 一阶导数在边界为零 |
| Smootherstep | C2 | `6x^5 - 15x^4 + 10x^3` | 一阶和二阶导数在边界为零 |

### Smoothstep 反转范围支持

当 `from_min > from_max` 时，Smoothstep 和 Smootherstep 模式会自动反转结果（`1 - smoothstep(max, min, value)`），确保映射方向正确。

### 向量版本的独立分量处理

向量版本对 x、y、z 三个分量独立执行相同的映射逻辑，每个分量可以有不同的源范围和目标范围。阶梯模式中，每个分量的步数也可以不同。

## 关联文件

- `kernel/svm/svm.h` — SVM 指令调度器
- `kernel/svm/clamp.h` — 简单的值钳位节点
- `kernel/svm/types.h` — `NODE_MAP_RANGE_LINEAR` 等模式常量定义
