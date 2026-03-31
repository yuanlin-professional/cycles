# radial_tiling.h - 径向平铺节点的SVM实现

## 概述

`radial_tiling.h` 实现了着色器虚拟机(SVM)中的径向平铺(Radial Tiling)节点。该节点将 2D 坐标空间围绕原点分割为多个扇形段（基于圆角多边形几何），并输出每个段内的局部坐标、段 ID 等信息。这是 Blender 4.x 新增的节点，用于创建基于极坐标的重复/平铺纹理图案。

## 核心函数

### `svm_node_radial_tiling`
- **签名**: `template<uint node_feature_mask> ccl_device_noinline int svm_node_radial_tiling(float *stack, uint4 node, int offset)`
- **功能**: 径向平铺节点的主函数。
- **输入参数**:
  - `vector` — 2D 输入坐标（使用 x, y 分量）
  - `r_gon_sides` — 多边形边数（正多边形的段数，最小值 2.0）
  - `r_gon_roundness` — 圆角度（0.0 = 尖角多边形，1.0 = 完全圆形），钳制到 [0, 1]
  - `normalize_r_gon_parameter` — 是否归一化段坐标参数
- **输出**:
  - `segment_coordinates` — 段内坐标 (float3，xy分量有效)
  - `segment_id` — 段序号 (float)
  - `max_unit_parameter` — 最大单位参数 (float)
  - `x_axis_A_angle_bisector` — X 轴到角平分线的角度 (float)
- **优化**: 通过 `stack_valid` 检查每个输出是否需要，只计算实际需要的输出值。
- **核心计算委托**: 调用 `radial_tiling_shared.h` 中定义的 `calculate_out_variables` 和 `calculate_out_segment_id` 函数。

### `RoundedPolygonStackOffsets`
- **功能**: 结构体，保存所有输入/输出参数在 SVM 栈中的偏移位置。

## 依赖关系

- **内部头文件**:
  - `radial_tiling_shared.h` — 核心计算函数（圆角多边形几何算法）
- **被引用**: `kernel/svm/svm.h`
- **编译宏**: 定义 `ADAPT_TO_SVM` 宏后包含 `radial_tiling_shared.h`，使共享代码适配 SVM 环境。

## 实现细节 / 关键算法

1. **SVM 适配宏**: 在包含 `radial_tiling_shared.h` 前定义 `ADAPT_TO_SVM` 宏，包含后取消定义。由于 SVM 代码已经使用了共享代码中的数学函数名（如 `cosf`、`sinf` 等），SVM 是共享实现的基础版本，不需要宏重映射。

2. **按需计算**: 通过 `stack_valid` 检查各输出插槽是否被连接，只计算实际需要的结果。例如，如果只连接了 `segment_id` 输出，则跳过较昂贵的 `calculate_out_variables` 调用。

3. **多边形段分割**: 坐标空间被分为 `r_gon_sides` 个等角扇形段，每段角度为 `2*pi / sides`。圆角参数控制段边界从直线（多边形边）到圆弧（圆形）的过渡。

## 关联文件

- `kernel/svm/radial_tiling_shared.h` — 共享计算实现
- `kernel/svm/svm.h` — SVM 主调度器
- `kernel/osl/shaders/node_radial_tiling.osl` — OSL 版本实现
