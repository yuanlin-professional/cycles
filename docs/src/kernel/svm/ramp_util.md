# ramp_util.h - CPU 端渐变查找工具函数

## 概述

`ramp_util.h` 提供了 CPU 端使用的颜色渐变和浮点渐变查找工具函数。与 `ramp.h` 中从 SVM 节点数组读取数据不同，本文件中的函数直接通过内存指针访问渐变数据，主要用于场景层着色器节点编译阶段的预览和求值。两者的查找算法逻辑完全一致，需保持同步。

## 核心函数

### `rgb_ramp_lookup`

```c
ccl_device_inline float3 rgb_ramp_lookup(const float3 *ramp, float f,
                                         bool interpolate, bool extrapolate,
                                         const int table_size)
```

- **功能**: 从 float3 指针数组中查找 RGB 颜色渐变值。
- **参数**:
  - `ramp`: 颜色渐变表指针（float3 数组）。
  - `f`: 查找位置，标准范围 [0, 1]。
  - `interpolate`: 是否在采样点之间线性插值。
  - `extrapolate`: 是否在范围外线性外推。
  - `table_size`: 表大小。
- **返回值**: float3 颜色值。
- **注意**: 此版本返回 float3（不含 Alpha），与 `ramp.h` 中返回 float4 的版本不同。

### `float_ramp_lookup`

```c
ccl_device float float_ramp_lookup(const float *ramp, float f,
                                   bool interpolate, bool extrapolate,
                                   const int table_size)
```

- **功能**: 从 float 指针数组中查找浮点渐变值。
- **参数**: 与 `rgb_ramp_lookup` 类似，但表为 float 数组。
- **返回值**: 单个浮点值。

## 依赖关系

- **内部头文件**:
  - `util/math.h` — 基础数学函数（clamp、float_to_int 等）
  - `util/types.h` — float3 等类型定义
- **被引用**:
  - `scene/shader_nodes.cpp` — 场景层着色器节点，用于 CPU 端的渐变求值

## 实现细节 / 关键算法

### 与 ramp.h 的一致性要求

文件开头注释明确指出：`svm_ramp.h`、`svm_ramp_util.h` 和 `node_ramp_util.h` 三个文件必须保持一致。它们实现相同的查找算法，区别仅在于数据访问方式：

| 文件 | 数据来源 | 使用场景 |
|------|---------|---------|
| `ramp.h` | SVM 节点数组（`fetch_float` / `fetch_node_float`） | GPU/CPU 内核执行 |
| `ramp_util.h` | 内存指针（`ramp[i]`） | CPU 场景层求值 |

### 外推算法

与 `ramp.h` 完全相同：
- `f < 0`: 使用 `ramp[0]` 和 `ramp[1]` 的斜率向负方向外推。
- `f > 1`: 使用 `ramp[table_size-1]` 和 `ramp[table_size-2]` 的斜率向正方向外推。
- 外推公式：`result = edge_value + slope * |f_out_of_range| * (table_size - 1)`

### NaN 安全

使用 `clamp(float_to_int(f), 0, table_size - 1)` 确保即使输入为 NaN 也不会越界访问数组。

## 关联文件

- `kernel/svm/ramp.h` — SVM 内核端的渐变查找（从 SVM 节点数组读取）
- `scene/shader_nodes.cpp` — 使用本文件函数的场景层代码
- `scene/node_ramp_util.h` — 另一个需要保持同步的相关文件
