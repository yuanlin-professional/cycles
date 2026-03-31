# wireframe.h - 线框节点的SVM实现

## 概述

`wireframe.h` 实现了着色器虚拟机(SVM)中的线框(Wireframe)节点。该节点通过计算着色点到三角形各边的距离，在三角形边缘附近输出 1.0，否则输出 0.0，从而在着色阶段生成线框效果。该实现改编自 Open Shading Language (OSL) 的线框着色器代码。

## 核心函数

### `wireframe`
- **签名**: `ccl_device_inline float wireframe(KernelGlobals kg, ShaderData *sd, const differential3 dP, const float size, const int pixel_size, float3 *P)`
- **功能**: 核心线框计算函数。
- **算法**:
  1. 仅对三角形图元有效（跳过毛发/点云）
  2. 获取三角形顶点坐标（支持运动模糊三角形 `motion_triangle_vertices`）
  3. 将顶点变换到世界空间（如需要）
  4. 如果启用像素尺寸模式：
     - 将 `dP.dx`/`dP.dy` 投影到观察平面（减去沿视线方向的分量）
     - 计算像素宽度 `pixelwidth = (|dP_projected_x| + |dP_projected_y|) / 2`
  5. 计算半宽度的平方: `(0.5 * size * pixelwidth)^2`
  6. 对每条边：
     - 计算 P 到边的叉积 `crs = cross(edge, P - vertex)`
     - 判断 `|crs|^2 / |edge|^2 < pixelwidth^2`（即距离的平方小于阈值）
     - 如果条件成立，返回 1.0
  7. 否则返回 0.0

### `svm_node_wireframe`
- **签名**: `ccl_device_noinline void svm_node_wireframe(KernelGlobals kg, ShaderData *sd, float *stack, const uint4 node)`
- **功能**: SVM 节点入口函数。
- **参数处理**:
  - `in_size` — 线宽（从栈加载）
  - `use_pixel_size` — 是否使用像素空间尺寸
  - `bump_offset` — 凹凸偏移方向（`NODE_BUMP_OFFSET_DX`/`DY`）
  - `bump_filter_width` — 凹凸滤波宽度
- **凹凸支持**: 根据 `bump_offset` 将位置 P 在 dP.dx 或 dP.dy 方向上偏移

## 依赖关系

- **内部头文件**:
  - `kernel/geom/motion_triangle.h` — 运动模糊三角形顶点
  - `kernel/geom/object.h` — 物体空间变换
  - `kernel/geom/triangle.h` — 三角形顶点获取
  - `kernel/svm/util.h` — SVM 栈操作工具
  - `kernel/util/differential.h` — 微分类型
  - `util/math_base.h` — 基础数学函数
- **被引用**: `kernel/svm/svm.h`

## 实现细节 / 关键算法

1. **点到直线距离**: 使用叉积方法计算点到边的距离。对于边 `AB` 和点 `P`，`cross(B-A, P-A)` 的模等于 `|AB| * distance(P, line_AB)`。因此 `|cross|^2 / |edge|^2 = distance^2`。

2. **半宽度处理**: 使用 `0.5 * size` 是因为相邻三角形会各渲染线框的一半宽度，两个三角形的边缘效果叠加后得到完整的线宽。

3. **像素空间尺寸**: 启用 `pixel_size` 时，线宽单位为像素而非世界空间。通过将位置微分投影到垂直于视线的平面上来估算像素尺寸。这使得线框在屏幕上保持恒定的视觉宽度。

4. **凹凸贴图支持**: 通过 `bump_offset` 参数，节点可以在微分偏移的位置上评估线框值，使得线框节点可以用作凹凸贴图的输入。

5. **BSD 许可证**: 该文件源自 Sony Pictures Imageworks 的 OSL 实现，使用 BSD-3-Clause 许可证（与其他文件的 Apache-2.0 不同）。

## 关联文件

- `kernel/svm/svm.h` — SVM 主调度器
- `kernel/geom/triangle.h` — 三角形顶点数据
