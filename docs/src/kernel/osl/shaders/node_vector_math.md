# node_vector_math.osl - 向量数学运算着色器

## 概述

该着色器实现了一个通用的向量数学运算节点，通过 `math_type` 字符串参数选择具体的运算类型。支持超过 25 种向量运算，涵盖基本向量算术、几何运算、逐分量运算和三角函数等。部分运算输出标量值（通过 `Value` 输出），其余输出向量（通过 `Vector` 输出）。依赖 `node_math.h` 头文件中的辅助函数。

## 着色器签名

```osl
shader node_vector_math(string math_type = "add",
                        vector Vector1 = vector(0.0, 0.0, 0.0),
                        vector Vector2 = vector(0.0, 0.0, 0.0),
                        vector Vector3 = vector(0.0, 0.0, 0.0),
                        float Scale = 1.0,
                        output float Value = 0.0,
                        output vector Vector = vector(0.0, 0.0, 0.0))
```

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| math_type | string | "add" | 运算类型标识符（详见下方运算列表） |
| Vector1 | vector | (0, 0, 0) | 第一个向量操作数 |
| Vector2 | vector | (0, 0, 0) | 第二个向量操作数（部分运算不使用） |
| Vector3 | vector | (0, 0, 0) | 第三个向量操作数（仅部分运算使用） |
| Scale | float | 1.0 | 标量参数（仅 `"scale"` 和 `"refract"` 运算使用） |

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| Value | float | 标量输出（仅 `dot_product`、`distance`、`length` 使用） |
| Vector | vector | 向量输出（大部分运算使用） |

## 实现逻辑

### 基本向量算术（输出 Vector）
| math_type | 运算 | 公式 |
|-----------|------|------|
| `"add"` | 向量加法 | Vector1 + Vector2 |
| `"subtract"` | 向量减法 | Vector1 - Vector2 |
| `"multiply"` | 逐分量乘法 | Vector1 * Vector2 |
| `"divide"` | 安全逐分量除法 | safe_divide(Vector1, Vector2) |
| `"scale"` | 标量缩放 | Vector1 * Scale |
| `"multiply_add"` | 乘加 | Vector1 * Vector2 + Vector3 |

### 几何运算
| math_type | 运算 | 公式 | 输出 |
|-----------|------|------|------|
| `"cross_product"` | 叉积 | cross(Vector1, Vector2) | Vector |
| `"dot_product"` | 点积 | dot(Vector1, Vector2) | Value |
| `"project"` | 投影 | project(Vector1, Vector2) | Vector |
| `"reflect"` | 反射 | reflect(Vector1, normalize(Vector2)) | Vector |
| `"refract"` | 折射 | refract(Vector1, normalize(Vector2), Scale) | Vector |
| `"faceforward"` | 面朝向 | compatible_faceforward(V1, V2, V3) | Vector |
| `"distance"` | 距离 | distance(Vector1, Vector2) | Value |
| `"length"` | 长度 | length(Vector1) | Value |
| `"normalize"` | 归一化 | normalize(Vector1) | Vector |

### 逐分量运算（输出 Vector）
| math_type | 运算 | 公式 |
|-----------|------|------|
| `"snap"` | 对齐 | snap(Vector1, Vector2) |
| `"floor"` | 向下取整 | floor(Vector1) |
| `"ceil"` | 向上取整 | ceil(Vector1) |
| `"modulo"` | 取模 | fmod(Vector1, Vector2) |
| `"wrap"` | 循环映射 | wrap(Vector1, Vector2, Vector3) |
| `"fraction"` | 小数部分 | Vector1 - floor(Vector1) |
| `"absolute"` | 绝对值 | abs(Vector1) |
| `"power"` | 逐分量幂 | pow(Vector1, Vector2) |
| `"sign"` | 符号 | sign(Vector1) |
| `"minimum"` | 逐分量最小 | min(Vector1, Vector2) |
| `"maximum"` | 逐分量最大 | max(Vector1, Vector2) |

### 逐分量三角函数（输出 Vector）
| math_type | 运算 | 公式 |
|-----------|------|------|
| `"sine"` | 正弦 | sin(Vector1) |
| `"cosine"` | 余弦 | cos(Vector1) |
| `"tangent"` | 正切 | tan(Vector1) |

## 对应 SVM 节点

对应 Cycles SVM 中的 `NODE_VECTOR_MATH` 节点。该节点在 Blender 节点编辑器中显示为"向量运算"（Vector Math）节点。
