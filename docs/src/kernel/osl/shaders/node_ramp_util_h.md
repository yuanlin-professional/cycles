# node_ramp_util.h - 渐变查找辅助函数头文件

## 概述

该头文件为 Cycles 渲染器的开放着色语言（OSL）着色器提供颜色渐变（Ramp）查找功能。实现了两个重载版本的 `rgb_ramp_lookup()` 函数，分别用于颜色类型和浮点类型的渐变查找。被 `node_rgb_ramp.osl`、`node_rgb_curves.osl` 和 `node_vector_curves.osl` 等着色器引用。

**一致性注意**：源文件注释指出 `svm_ramp.h`、`svm_ramp_util.h` 和 `node_ramp_util.h` 三个文件必须保持实现一致。

## 函数列表

| 函数签名 | 说明 |
|----------|------|
| `color rgb_ramp_lookup(color ramp[], float at, int interpolate, int extrapolate)` | 颜色渐变查找 |
| `float rgb_ramp_lookup(float ramp[], float at, int interpolate, int extrapolate)` | 浮点渐变查找 |

## 函数参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| ramp | color[] 或 float[] | 渐变查找表数组 |
| at | float | 查找位置（通常在 [0, 1] 范围内） |
| interpolate | int | 是否在采样点间线性插值。1 = 插值，0 = 最近邻 |
| extrapolate | int | 是否在范围外外推。1 = 启用外推，0 = 钳制到边界 |

## 实现逻辑

两个重载版本的实现逻辑完全相同，仅操作的数据类型不同：

### 外推处理（当 `extrapolate == 1` 且 `at` 超出 [0, 1]）

1. **下溢外推**（`at < 0`）：
   - 取查找表前两个元素：`t0 = ramp[0]`，差分 `dy = ramp[0] - ramp[1]`。
   - 外推结果：`t0 + dy * (-at) * (table_size - 1)`。

2. **上溢外推**（`at > 1`）：
   - 取查找表后两个元素：`t0 = ramp[last]`，差分 `dy = ramp[last] - ramp[last-1]`。
   - 外推结果：`t0 + dy * (at - 1) * (table_size - 1)`。

外推使用线性延伸，基于边界处的斜率，乘以 `(table_size - 1)` 用于正确缩放。

### 标准查找（在 [0, 1] 范围内或外推禁用）

1. 将 `at` 钳制到 [0, 1] 范围并缩放到表索引：`f = clamp(at, 0, 1) * (table_size - 1)`。
2. 计算整数索引 `i = (int)f` 和小数部分 `t = f - i`。
3. 对索引进行额外的安全钳制，防止 NaN 值导致的越界。
4. 取基础结果：`result = ramp[i]`。
5. **插值**（当 `interpolate == 1` 且 `t > 0`）：
   - `result = (1 - t) * result + t * ramp[i + 1]`（线性插值）。
6. 返回结果。

## 关键设计特点

1. **NaN 安全**：对整数索引做额外边界检查（`i < 0` 或 `i >= table_size`），因为 NaN 的浮点到整数转换可能产生任意值。
2. **双重钳制**：既钳制浮点值也钳制整数索引，确保在所有边界情况下都不会越界访问。
3. **外推缩放**：外推距离乘以 `(table_size - 1)` 是因为表是按归一化坐标均匀分布的，每个表项间距为 `1 / (table_size - 1)`。
