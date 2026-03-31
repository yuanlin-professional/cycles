# math_util.h - 数学运算核心实现与物理颜色计算工具

## 概述

`math_util.h` 是 Cycles SVM 数学运算的核心实现文件，包含标量数学函数 `svm_math()`、向量数学函数 `svm_vector_math()` 以及多个物理颜色计算工具函数（黑体辐射、伽马校正、波长转颜色）。该文件既服务于数学节点，也被黑体节点、波长节点、伽马节点等多个颜色相关节点所依赖。

## 核心函数

### `svm_math`

```c
ccl_device float svm_math(NodeMathType type, const float a, float b, const float c)
```

- **功能**: 根据 `NodeMathType` 类型执行标量数学运算。
- **支持的运算类型（共 30+ 种）**:
  - **算术运算**: 加(ADD)、减(SUBTRACT)、乘(MULTIPLY)、除(DIVIDE)、乘加(MULTIPLY_ADD)
  - **幂/对数**: 幂(POWER)、对数(LOGARITHM)、平方根(SQRT)、逆平方根(INV_SQRT)、指数(EXPONENT)
  - **取整**: 四舍五入(ROUND)、向下取整(FLOOR)、向上取整(CEIL)、截断(TRUNC)、小数部分(FRACTION)、对齐(SNAP)
  - **比较**: 小于(LESS_THAN)、大于(GREATER_THAN)、比较(COMPARE)
  - **三角函数**: sin、cos、tan、sinh、cosh、tanh、asin、acos、atan、atan2
  - **其他**: 绝对值(ABSOLUTE)、符号(SIGN)、弧度(RADIANS)、角度(DEGREES)、最小值(MINIMUM)、最大值(MAXIMUM)、模(MODULO)、向下取整模(FLOORED_MODULO)、包裹(WRAP)、乒乓(PINGPONG)、平滑最小(SMOOTH_MIN)、平滑最大(SMOOTH_MAX)

### `svm_vector_math`

```c
ccl_device void svm_vector_math(ccl_private float *value,
                                ccl_private float3 *vector,
                                NodeVectorMathType type,
                                const float3 a, const float3 b, const float3 c,
                                float param1)
```

- **功能**: 根据 `NodeVectorMathType` 类型执行向量数学运算。
- **支持的运算类型（共 27 种）**:
  - **向量输出**: 加、减、乘、除、叉积(CROSS_PRODUCT)、投影(PROJECT)、反射(REFLECT)、折射(REFRACT)、面向前方(FACEFORWARD)、乘加(MULTIPLY_ADD)、缩放(SCALE)、归一化(NORMALIZE)、对齐(SNAP)、取整系列、取模、包裹、绝对值、幂、符号、最小/最大、三角函数
  - **标量输出**: 点积(DOT_PRODUCT)、距离(DISTANCE)、长度(LENGTH)

### `svm_math_blackbody_color_rec709`

```c
ccl_device float3 svm_math_blackbody_color_rec709(const float t)
```

- **功能**: 根据色温（开尔文）计算 Rec.709 色彩空间下的黑体辐射颜色。
- **有效范围**: 800K - 12000K，使用分段多项式近似 `a/x + bx + c`（R、G 通道）和三次多项式（B 通道）。
- **特性**: 结果可能为负值（支持超出 Rec.709 色域的广色域），调用方需自行钳位。

### `svm_math_gamma_color`

```c
ccl_device_inline float3 svm_math_gamma_color(float3 color, const float gamma)
```

- **功能**: 对颜色逐通道应用伽马校正 `pow(color, gamma)`。
- **特殊处理**: gamma=0 时返回白色(1,1,1)；仅对正值通道执行幂运算，避免负数幂导致的 NaN。

### `svm_math_wavelength_color_xyz`

```c
ccl_device float3 svm_math_wavelength_color_xyz(const float lambda_nm)
```

- **功能**: 将可见光波长（纳米）转换为 CIE XYZ 色彩空间颜色值。
- **有效范围**: 380nm - 780nm（80 个采样点，步长 5nm），使用查表+线性插值。
- **超出范围**: 返回黑色(0,0,0)。

## 依赖关系

- **内部头文件**:
  - `kernel/svm/types.h` — `NodeMathType`、`NodeVectorMathType` 枚举定义
  - `kernel/tables.h` — 黑体辐射查找表 `blackbody_table_r/g/b` 和 CIE 色彩匹配表 `cie_color_match`
  - `util/math.h` — 基础数学函数（safe_divide、safe_powf 等安全数学运算）
  - `util/types.h` — float3 等类型定义
- **被引用**:
  - `kernel/svm/math.h` — 标量/向量数学节点入口
  - `kernel/svm/gamma.h` — 伽马节点（调用 `svm_math_gamma_color`）
  - `kernel/svm/blackbody.h` — 黑体节点（调用 `svm_math_blackbody_color_rec709`）
  - `kernel/svm/wavelength.h` — 波长节点（调用 `svm_math_wavelength_color_xyz`）
  - `kernel/svm/closure.h` — 闭包节点
  - `scene/shader_nodes.cpp` — 场景层着色器节点（CPU 端复用）

## 实现细节 / 关键算法

### 黑体辐射近似算法

使用 7 段分段近似，温度区间分别为 [800, 965)、[965, 1167)、[1167, 1449)、[1449, 1902)、[1902, 3315)、[3315, 6365)、[6365, 12000]。R 和 G 通道使用 `a/t + b*t + c` 形式的有理函数近似，B 通道使用三次多项式 `((a*t + b)*t + c)*t + d`。查找表存储在 `kernel/tables.h` 的 `blackbody_table_r/g/b` 中。

### 安全数学运算

所有可能产生除零、负数开方等异常的运算均使用 `safe_` 前缀版本（如 `safe_divide`、`safe_powf`、`safe_sqrtf`、`safe_logf`、`safe_asinf` 等），确保在 GPU 上不会产生 NaN 或 Inf。

### CIE 波长查表

波长转 XYZ 使用 CIE 1931 标准观察者色彩匹配函数，以 5nm 为步长存储 80 个采样点。通过线性插值获取任意波长对应的 XYZ 值。

## 关联文件

- `kernel/svm/math.h` — 数学节点 SVM 入口
- `kernel/svm/blackbody.h` — 黑体辐射节点
- `kernel/svm/wavelength.h` — 波长转 RGB 节点
- `kernel/svm/gamma.h` — 伽马校正节点
- `kernel/tables.h` — 预计算查找表数据
