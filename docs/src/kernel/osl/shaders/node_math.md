# node_math.osl - 数学运算着色器

## 概述

该着色器实现了一个通用的标量数学运算节点，通过 `math_type` 字符串参数选择具体的运算类型。支持超过 30 种数学运算，涵盖基本算术、三角函数、比较运算、取整运算等。依赖 `node_math.h` 头文件中定义的安全运算辅助函数。

## 着色器签名

```osl
shader node_math(string math_type = "add",
                 float Value1 = 0.5,
                 float Value2 = 0.5,
                 float Value3 = 0.5,
                 output float Value = 0.0)
```

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| math_type | string | "add" | 运算类型标识符（详见下方运算列表） |
| Value1 | float | 0.5 | 第一个操作数 |
| Value2 | float | 0.5 | 第二个操作数（部分运算不使用） |
| Value3 | float | 0.5 | 第三个操作数（仅部分运算使用） |

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| Value | float | 运算结果 |

## 实现逻辑

根据 `math_type` 选择对应的数学运算。支持的运算类型如下：

### 基本算术运算
| math_type | 运算 | 公式 | 使用参数 |
|-----------|------|------|----------|
| `"add"` | 加法 | Value1 + Value2 | V1, V2 |
| `"subtract"` | 减法 | Value1 - Value2 | V1, V2 |
| `"multiply"` | 乘法 | Value1 * Value2 | V1, V2 |
| `"divide"` | 安全除法 | safe_divide(Value1, Value2) | V1, V2 |
| `"multiply_add"` | 乘加 | Value1 * Value2 + Value3 | V1, V2, V3 |
| `"power"` | 幂运算 | pow(Value1, Value2) | V1, V2 |
| `"logarithm"` | 安全对数 | safe_log(Value1, Value2) | V1, V2 |
| `"sqrt"` | 安全平方根 | safe_sqrt(Value1) | V1 |
| `"inversesqrt"` | 平方根倒数 | inversesqrt(Value1) | V1 |
| `"absolute"` | 绝对值 | fabs(Value1) | V1 |
| `"sign"` | 符号函数 | sign(Value1) | V1 |
| `"exponent"` | 自然指数 | exp(Value1) | V1 |

### 比较运算
| math_type | 运算 | 公式 | 使用参数 |
|-----------|------|------|----------|
| `"minimum"` | 最小值 | min(Value1, Value2) | V1, V2 |
| `"maximum"` | 最大值 | max(Value1, Value2) | V1, V2 |
| `"less_than"` | 小于 | Value1 < Value2 ? 1.0 : 0.0 | V1, V2 |
| `"greater_than"` | 大于 | Value1 > Value2 ? 1.0 : 0.0 | V1, V2 |
| `"compare"` | 近似比较 | 差值 <= max(Value3, 1e-5) ? 1.0 : 0.0 | V1, V2, V3 |
| `"smoothmin"` | 平滑最小值 | smoothmin(Value1, Value2, Value3) | V1, V2, V3 |
| `"smoothmax"` | 平滑最大值 | -smoothmin(-V1, -V2, V3) | V1, V2, V3 |

### 取整运算
| math_type | 运算 | 公式 | 使用参数 |
|-----------|------|------|----------|
| `"round"` | 四舍五入 | floor(Value1 + 0.5) | V1 |
| `"floor"` | 向下取整 | floor(Value1) | V1 |
| `"ceil"` | 向上取整 | ceil(Value1) | V1 |
| `"trunc"` | 截断取整 | trunc(Value1) | V1 |
| `"fraction"` | 小数部分 | Value1 - floor(Value1) | V1 |
| `"modulo"` | 安全取模 | safe_modulo(Value1, Value2) | V1, V2 |
| `"floored_modulo"` | 向下取整取模 | safe_floored_modulo(Value1, Value2) | V1, V2 |
| `"snap"` | 对齐 | floor(V1 / V2) * V2 | V1, V2 |
| `"wrap"` | 循环映射 | wrap(Value1, Value2, Value3) | V1, V2, V3 |
| `"pingpong"` | 乒乓映射 | pingpong(Value1, Value2) | V1, V2 |

### 三角函数
| math_type | 运算 | 公式 | 使用参数 |
|-----------|------|------|----------|
| `"sine"` | 正弦 | sin(Value1) | V1 |
| `"cosine"` | 余弦 | cos(Value1) | V1 |
| `"tangent"` | 正切 | tan(Value1) | V1 |
| `"sinh"` | 双曲正弦 | sinh(Value1) | V1 |
| `"cosh"` | 双曲余弦 | cosh(Value1) | V1 |
| `"tanh"` | 双曲正切 | tanh(Value1) | V1 |
| `"arcsine"` | 反正弦 | asin(Value1) | V1 |
| `"arccosine"` | 反余弦 | acos(Value1) | V1 |
| `"arctangent"` | 反正切 | atan(Value1) | V1 |
| `"arctan2"` | 二参数反正切 | atan2(Value1, Value2) | V1, V2 |

### 角度转换
| math_type | 运算 | 公式 | 使用参数 |
|-----------|------|------|----------|
| `"radians"` | 角度转弧度 | radians(Value1) | V1 |
| `"degrees"` | 弧度转角度 | degrees(Value1) | V1 |

注意：开放着色语言（OSL）的 `asin`、`acos` 和 `pow` 函数默认安全，不会产生未定义行为。

## 对应 SVM 节点

对应 Cycles SVM 中的 `NODE_MATH` 节点。该节点在 Blender 节点编辑器中显示为"数学"（Math）节点。
