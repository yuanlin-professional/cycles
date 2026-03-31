# node_radial_tiling_shared.h - 径向平铺共享代码头文件

## 概述

该头文件包含径向平铺（Radial Tiling）节点的核心数学计算代码，是一个跨平台共享实现。同一份代码通过宏适配机制在四个不同的渲染后端中使用：几何节点（Geometry Nodes）、开放着色语言（OSL）、SVM 内核以及 GPU GLSL 着色器。该文件实现了将二维坐标进行正多边形径向分割和参数化的算法，约 1179 行。

## 宏定义/函数/类型

### 平台适配宏

文件顶部通过预处理器条件编译定义了四组数学函数宏映射：

| 适配目标 | 预处理宏 | 说明 |
|----------|----------|------|
| 几何节点 | `ADAPT_TO_GEOMETRY_NODES` | 映射到 `math::` 命名空间 |
| 开放着色语言（OSL） | `ADAPT_TO_OSL` | 映射到 OSL 内建函数名 |
| SVM | `ADAPT_TO_SVM` | 无需适配（基准实现） |
| GLSL（默认） | 无特定宏 | 映射到 GLSL 内建函数名 |

适配的数学函数包括：`atanf`、`atan2f`、`ceilf`、`cosf`、`fabsf`、`floorf`、`fmaxf`、`fminf`、`fractf`、`mix`、`sinf`、`sqrtf`、`sqr`、`tanf` 等。

OSL 适配还额外定义了类型别名：`bool` -> `int`、`float2` -> `vector2`、`float4` -> `vector4`、`false` -> `0`、`true` -> `1`。

### 核心函数

| 函数名 | 返回类型 | 说明 |
|--------|----------|------|
| `calculate_out_variables_full_roundness_irregular_circular` | float4 | 计算完整圆角度的不规则圆形径向平铺输出变量（仅几何节点可用） |
| `calculate_out_variables_irregular_circular` | float4 | 计算不规则圆形径向平铺的输出变量 |
| `calculate_out_variables` | float4 | 计算一般径向平铺的输出变量，支持正/不规则多边形和圆形模式 |
| `calculate_out_segment_id` | float | 计算坐标所在的正多边形段 ID |

### 命名约定

文件使用了特定的命名规范：
- `l_x` = `length_x`（X 的长度）
- `x_A_y` = `x_Angle_y`（从 X 到 Y 的逆时针无符号角度）
- `x_SA_y` = `x_SignedAngle_y`（从 X 到 Y 的有符号角度）
- `x_MSA_y` = `x_MirroredSignedAngle_y`（镜像有符号角度）
- `z_R_w` = `z_Ratio_w`（Z 与 W 的比率）

### 条件编译控制宏

| 宏名 | 说明 |
|------|------|
| `ONLY_CHECK_IN_GEOMETRY_NODES_IMPLEMENTATION(X)` | 仅在几何节点实现中执行检查，其他平台始终为 true |

## 依赖关系

- **同步文件**：以下四个文件必须始终保持完全一致的内容：
  - `radial_tiling_shared.hh`（C++ 头文件）
  - `node_radial_tiling_shared.h`（OSL 头文件）
  - `radial_tiling_shared.h`（SVM 头文件）
  - `gpu_shader_material_radial_tiling_shared.glsl`（GLSL 着色器）
- **被依赖**：`node_radial_tiling.osl`（径向平铺着色器节点）
- **许可证**：Apache-2.0，Blender Authors 2024-2025
