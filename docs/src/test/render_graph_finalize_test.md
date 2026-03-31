# render_graph_finalize_test.cpp - 着色器图优化与常量折叠测试

## 概述

此文件是 Cycles 渲染器测试套件中最大的测试文件之一，全面测试着色器图（Shader Graph）的最终化和优化过程。主要覆盖以下优化策略：节点去重（deduplication）、常量折叠（constant folding）、部分常量折叠和随机采样优化（stochastic sampling）。测试使用自定义的 `ShaderGraphBuilder` 辅助类构建着色器图，然后验证优化后的图形结构是否符合预期。

## 辅助构建器

### ShaderNodeBuilder<T>
- **功能**: 用于创建和配置着色器节点的模板构建器，支持链式调用 `set()` 和 `set_param()` 方法

### ShaderGraphBuilder
- **功能**: 着色器图构建器，提供节点添加（`add_node`）、连接（`add_connection`）和期望输出设置（`output_color`/`output_closure`）等接口

### RenderGraph (TEST_F 夹具)
- **功能**: 测试夹具类，负责初始化 `Scene`、`ShaderGraph`，提供 `graph.finalize()` 调用和验证方法

## 测试用例

### TEST_F(RenderGraph, deduplicate_deep)
- **功能**: 验证着色器图中深层重复节点的去重优化。构建包含多层相同节点的图，确认优化后重复节点被合并。

### 常量折叠 - 单节点测试
| 测试名 | 功能 |
|--------|------|
| `constant_fold_rgb_to_bw` | RGB 转亮度值的常量折叠 |
| `constant_fold_emission1` | 自发光着色器折叠（颜色为零） |
| `constant_fold_emission2` | 自发光着色器折叠（强度为零） |
| `constant_fold_background1` | 背景着色器折叠（颜色为零） |
| `constant_fold_background2` | 背景着色器折叠（强度为零） |
| `constant_fold_shader_add` | 着色器加法折叠 |
| `constant_fold_shader_mix` | 着色器混合折叠 |
| `constant_fold_invert` | 颜色反转折叠 |
| `constant_fold_invert_fac_0` | 反转节点因子为 0 的折叠 |
| `constant_fold_invert_fac_0_const` | 反转节点因子为 0（常量输入）的折叠 |
| `constant_fold_gamma` | Gamma 校正折叠 |
| `constant_fold_bright_contrast` | 亮度/对比度折叠 |
| `constant_fold_blackbody` | 黑体辐射折叠 |

### 常量折叠 - Mix 节点测试
| 测试名 | 功能 |
|--------|------|
| `constant_fold_mix_add` | 混合(加法)常量折叠 |
| `constant_fold_mix_add_clamp` | 混合(加法+钳制)常量折叠 |
| `constant_fold_part_mix_dodge_no_fac_0` | 减淡混合因子非零的部分折叠 |
| `constant_fold_part_mix_light_no_fac_0` | 线性光混合因子非零的部分折叠 |
| `constant_fold_part_mix_burn_no_fac_0` | 加深混合因子非零的部分折叠 |
| `constant_fold_part_mix_blend_clamped_no_fac_0` | 混合(钳制)因子非零的部分折叠 |
| `constant_fold_part_mix_blend` | 混合的部分折叠 |
| `constant_fold_part_mix_sub_same_fac_bad` | 混合(减法)相同因子错误情况 |
| `constant_fold_part_mix_sub_same_fac_1` | 混合(减法)相同因子为 1 |
| `constant_fold_part_mix_add_0` | 混合(加法)第二输入为 0 |
| `constant_fold_part_mix_sub_0` | 混合(减法)第二输入为 0 |
| `constant_fold_part_mix_mul_1` | 混合(乘法)第二输入为 1 |
| `constant_fold_part_mix_div_1` | 混合(除法)第二输入为 1 |
| `constant_fold_part_mix_mul_0` | 混合(乘法)第二输入为 0 |
| `constant_fold_part_mix_div_0` | 混合(除法)第二输入为 0 |

### 常量折叠 - 数学节点测试
| 测试名 | 功能 |
|--------|------|
| `constant_fold_math` | 数学节点常量折叠 |
| `constant_fold_math_clamp` | 数学节点带钳制的常量折叠 |
| `constant_fold_part_math_add_0` | 数学加法第二输入为 0 |
| `constant_fold_part_math_sub_0` | 数学减法第二输入为 0 |
| `constant_fold_part_math_mul_1` | 数学乘法第二输入为 1 |
| `constant_fold_part_math_div_1` | 数学除法第二输入为 1 |
| `constant_fold_part_math_mul_0` | 数学乘法第二输入为 0 |
| `constant_fold_part_math_div_0` | 数学除法第二输入为 0 |
| `constant_fold_part_math_pow_0` | 数学幂运算指数为 0 |
| `constant_fold_part_math_pow_1` | 数学幂运算指数为 1 |

### 常量折叠 - 向量数学节点测试
| 测试名 | 功能 |
|--------|------|
| `constant_fold_vector_math` | 向量数学节点常量折叠 |
| `constant_fold_part_vecmath_add_0` | 向量加法第二输入为零向量 |
| `constant_fold_part_vecmath_sub_0` | 向量减法第二输入为零向量 |
| `constant_fold_part_vecmath_cross_0` | 向量叉乘第二输入为零向量 |

### 常量折叠 - 其他节点测试
| 测试名 | 功能 |
|--------|------|
| `constant_fold_separate_combine_rgb` | 分离/合并 RGB 的常量折叠 |
| `constant_fold_separate_combine_xyz` | 分离/合并 XYZ 的常量折叠 |
| `constant_fold_separate_combine_hsv` | 分离/合并 HSV 的常量折叠 |
| `constant_fold_gamma_part_0` | Gamma 部分折叠（指数为 0） |
| `constant_fold_gamma_part_1` | Gamma 部分折叠（指数为 1） |
| `constant_fold_bump` | Bump 节点折叠 |
| `constant_fold_bump_no_input` | Bump 节点无输入时的折叠 |
| `constant_fold_rgb_curves` | RGB 曲线节点折叠 |
| `constant_fold_rgb_curves_fac_0` | RGB 曲线因子为 0 的折叠 |
| `constant_fold_rgb_curves_fac_0_const` | RGB 曲线因子为 0（常量）的折叠 |
| `constant_fold_vector_curves` | 向量曲线节点折叠 |
| `constant_fold_vector_curves_fac_0` | 向量曲线因子为 0 的折叠 |
| `constant_fold_rgb_ramp` | RGB 渐变节点折叠 |
| `constant_fold_rgb_ramp_flat` | RGB 渐变（平坦）节点折叠 |

### 常量折叠 - 类型转换测试
| 测试名 | 功能 |
|--------|------|
| `constant_fold_convert_float_color_float` | float -> color -> float 转换折叠 |
| `constant_fold_convert_color_vector_color` | color -> vector -> color 转换折叠 |
| `constant_fold_convert_color_float_color` | color -> float -> color 转换折叠 |

### 随机采样优化测试
| 测试名 | 功能 |
|--------|------|
| `stochastic_sample_math_multiply` | 数学乘法节点的随机采样优化 |
| `not_stochastic_sample_math_power` | 数学幂运算不应进行随机采样优化 |
| `stochastic_sample_principled_volume_mix` | Principled Volume 混合的随机采样优化 |

## 依赖关系
- **被测源文件**: `scene/shader_graph.h`, `scene/shader_nodes.h`
- **测试框架**: Google Test (GTest)，使用 `TEST_F` 夹具
- **辅助依赖**: `device/device.h`, `scene/colorspace.h`, `scene/scene.h`, `util/array.h`, `util/log.h`, `util/stats.h`, `util/string.h`, `util/vector.h`

## 关联文件
- `src/scene/shader_graph.h` - 着色器图定义
- `src/scene/shader_graph.cpp` - 着色器图实现（含 `finalize()` 方法）
- `src/scene/shader_nodes.h` - 着色器节点定义
