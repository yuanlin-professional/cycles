# radial_tiling_shared.h - 径向平铺节点的跨平台共享计算实现

## 概述

`radial_tiling_shared.h` 是径向平铺(Radial Tiling)节点的核心数学计算实现文件。该文件被设计为跨多个渲染后端共享的代码，通过条件编译宏适配不同的目标平台：SVM（着色器虚拟机）、OSL（Open Shading Language）、GLSL（GPU 着色器）和几何节点(Geometry Nodes)。它实现了圆角多边形的几何计算，包括段坐标参数化、段 ID 计算等核心算法。

**重要**: 以下四个文件必须始终保持内容一致（是彼此的精确副本）：
- `radial_tiling_shared.hh`（几何节点版本）
- `node_radial_tiling_shared.h`（OSL 版本）
- `radial_tiling_shared.h`（SVM 版本 — 本文件）
- `gpu_shader_material_radial_tiling_shared.glsl`（GLSL 版本）

## 核心函数

### `calculate_out_variables_full_roundness_irregular_circular`（仅几何节点）
- **签名**: `ccl_device float4 calculate_out_variables_full_roundness_irregular_circular(bool calculate_r_gon_parameter_field, bool normalize_r_gon_parameter, float r_gon_sides, float2 coord, float l_coord)`
- **功能**: 处理完全圆角(`roundness=1.0`)且非整数边数的特殊情况。仅在几何节点实现中启用（`ADAPT_TO_GEOMETRY_NODES`），因为它是性能优化的特化版本。

### `calculate_out_variables_irregular_circular`
- **签名**: `ccl_device float4 calculate_out_variables_irregular_circular(bool calculate_r_gon_parameter_field, bool calculate_max_unit_parameter, bool normalize_r_gon_parameter, float r_gon_sides, float r_gon_roundness, float2 coord, float l_coord)`
- **功能**: 处理非整数边数（不规则多边形）的通用圆角情况。
- **算法**: 将空间分为三大区域：
  1. **常规区域**（落在整数段内）：进一步分为直线部分和圆角部分
  2. **不规则区域**（跨越整数边界的最后一段）：分为不规则直线部分和不规则圆角部分
  - 每个区域计算 `l_angle_bisector`（到角平分线的径向距离比）和 `r_gon_parameter`（段内横向参数）

### `calculate_out_variables`
- **签名**: `ccl_device float4 calculate_out_variables(bool calculate_r_gon_parameter_field, bool calculate_max_unit_parameter, bool normalize_r_gon_parameter, float r_gon_sides, float r_gon_roundness, float2 coord)`
- **功能**: 顶层输出变量计算函数，是外部调用的主入口。
- **返回**: `float4(l_angle_bisector, r_gon_parameter, max_unit_parameter, x_axis_A_angle_bisector)`
- **分发逻辑**:
  1. 计算 `l_coord`（输入坐标到原点的距离）
  2. 如果边数为整数（`fract(sides) == 0`）：使用常规多边形路径
  3. 如果边数非整数且圆角度为 1.0：使用完全圆角特化路径（仅几何节点）
  4. 否则：使用 `calculate_out_variables_irregular_circular` 通用路径

### `calculate_out_segment_id`
- **签名**: `ccl_device float calculate_out_segment_id(float r_gon_sides, float2 coord)`
- **功能**: 计算输入坐标所在的段序号。
- **算法**: `floor(atan2(y,x) + (y<0)*2pi) / (2pi/sides))`

## 依赖关系

- **内部头文件**: 无（自包含，通过宏适配各平台）
- **被引用**: `kernel/svm/radial_tiling.h`（通过 `#include "radial_tiling_shared.h"` 包含）

## 实现细节 / 关键算法

### 代码适配宏系统

文件头部定义了四套宏映射：

| 目标平台 | 适配宏 | 数学函数映射 |
|---------|--------|------------|
| 几何节点 | `ADAPT_TO_GEOMETRY_NODES` | `cosf -> math::cos` 等 |
| OSL | `ADAPT_TO_OSL` | `cosf -> cos`, `float2 -> vector2` 等 |
| SVM | `ADAPT_TO_SVM` | 无需映射（SVM 是基础版本） |
| GLSL | 默认 | `cosf -> cos`, `make_float2 -> float2` 等 |

### 命名约定

- `l_x` — 向量 X 的长度(length)
- `x_A_y` — 从 X 到 Y 的逆时针无符号角度(Angle)，范围 [0, 2pi]
- `x_SA_y` — 从 X 到 Y 的有符号角度(Signed Angle)，范围 [-pi, pi]
- `x_MSA_y` — 沿某向量镜像后的有符号角度(Mirrored Signed Angle)
- `x_R_y` — 比值 x/y (Ratio)

### 圆角多边形几何

1. **段分割器(Segment Divider)**: 多边形的每条边（或边的延长线）将空间分为扇形段
2. **角平分线(Angle Bisector)**: 相邻两条段分割器之间的平分线
3. **倒角起点(Bevel Start)**: 圆角部分开始的角度位置，由 `roundness` 参数控制
4. **圆弧圆心**: 用于圆角过渡的圆弧中心，通过几何关系计算

### 不规则段处理

当边数为非整数（如 5.3）时，最后一段（从最后一条完整边到回绕的第一条边）比常规段窄。代码将这种不规则段作为特殊情况处理，使用独立的圆弧参数来平滑过渡。

### 几何节点特化优化

`ONLY_CHECK_IN_GEOMETRY_NODES_IMPLEMENTATION(X)` 宏在几何节点实现中展开为参数 `(X)`（即条件检查），在其他实现中展开为 `true`（即始终执行）。这使得几何节点版本可以在运行时根据需要跳过不必要的计算，而 SVM/OSL/GLSL 版本总是计算所有值以避免分支开销。

## 关联文件

- `kernel/svm/radial_tiling.h` — SVM 节点入口
- `kernel/osl/shaders/node_radial_tiling.osl` — OSL 版本
- 几何节点版本: `radial_tiling_shared.hh`
- GLSL 版本: `gpu_shader_material_radial_tiling_shared.glsl`
