# util_float8_test.h - float8 向量类型共享测试定义

## 概述

此头文件定义了 `vfloat8`（8 元素浮点向量）类型的全部测试用例，被 `util_float8_avx2_test.cpp`、`util_float8_avx_test.cpp` 和 `util_float8_sse2_test.cpp` 三个源文件包含。通过宏 `TEST_CATEGORY_NAME` 区分不同指令集后端，使同一套测试可在 SSE2、AVX 和 AVX2 三种实现上分别运行。

## CPU 能力检测

每个测试开头调用 `INIT_FLOAT8_TEST` 宏，该宏内部通过 `validate_cpu_capabilities()` 检测当前 CPU 是否支持编译时指定的指令集（AVX2 > AVX > SSE4.2）。若不支持则跳过测试。

## 测试辅助数据

| 函数/变量 | 值 | 说明 |
|-----------|------|------|
| `float8_a()` | `{0.1, 0.2, 0.3, 0.4, 0.5, 0.6, 0.7, 0.8}` | 测试向量 A |
| `float8_b()` | `{1.0, 2.0, 3.0, 4.0, 5.0, 6.0, 7.0, 8.0}` | 测试向量 B |
| `float8_c()` | `{1.1, 2.2, 3.3, 4.4, 5.5, 6.6, 7.7, 8.8}` | 测试向量 C |
| `float_b` | `1.5f` | 标量测试值 |

## 测试辅助宏

| 宏名称 | 说明 |
|--------|------|
| `compare_vector_scalar(a, b)` | 验证向量每个分量等于标量 |
| `compare_vector_vector(a, b)` | 验证两个向量逐分量相等 |
| `compare_vector_vector_near(a, b, err)` | 验证两个向量逐分量近似相等 |
| `basic_test_vv(a, b, op)` | 向量-向量运算测试 |
| `basic_test_vf(a, b, op)` | 向量-标量运算测试 |

## 测试用例

### 基本算术运算 - 向量与向量
| 测试名 | 功能 |
|--------|------|
| `float8_add_vv` | 向量加法 `a + b`，逐分量验证 |
| `float8_sub_vv` | 向量减法 `a - b`，逐分量验证 |
| `float8_mul_vv` | 向量乘法 `a * b`，逐分量验证 |
| `float8_div_vv` | 向量除法 `a / b`，逐分量验证 |

### 基本算术运算 - 向量与标量
| 测试名 | 功能 |
|--------|------|
| `float8_add_vf` | 向量加标量 `a + 1.5`，逐分量验证 |
| `float8_sub_vf` | 向量减标量 `a - 1.5`，逐分量验证 |
| `float8_mul_vf` | 向量乘标量 `a * 1.5`，逐分量验证 |
| `float8_div_vf` | 向量除标量 `a / 1.5`，逐分量验证 |

### 构造与特殊函数
| 测试名 | 功能 |
|--------|------|
| `float8_ctor` | 构造函数测试：分别验证 8 参数构造和单值广播构造 |
| `float8_sqrt` | 平方根运算，验证 `sqrt({1,4,9,16,25,36,49,64}) == {1,2,3,4,5,6,7,8}` |
| `float8_min_max` | `min`/`max` 函数，验证逐分量取最小/最大值 |
| `float8_shuffle` | 分量重排（shuffle）操作，验证三种不同的重排模式 |

## 依赖关系
- **被测源文件**: `util/types.h`, `util/math.h`, `util/system.h`
- **测试框架**: Google Test (GTest)

## 关联文件
- `src/test/util_float8_avx2_test.cpp` - AVX2 指令集测试入口
- `src/test/util_float8_avx_test.cpp` - AVX 指令集测试入口
- `src/test/util_float8_sse2_test.cpp` - SSE2 指令集测试入口
- `src/util/types.h` - vfloat8 类型定义
