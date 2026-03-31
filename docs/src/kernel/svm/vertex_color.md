# vertex_color.h - 顶点颜色节点的SVM实现

## 概述

`vertex_color.h` 实现了着色器虚拟机(SVM)中的顶点颜色(Vertex Color / Color Attribute)节点。该节点从网格的顶点颜色属性层中读取颜色和 Alpha 值，支持 float3 和 float4/RGBA 两种存储格式，并提供凹凸贴图(bump mapping)所需的微分偏移变体。

## 核心函数

### `svm_node_vertex_color`
- **签名**: `ccl_device_noinline void svm_node_vertex_color(KernelGlobals kg, ShaderData *sd, float *stack, const uint4 node)`
- **功能**: 读取顶点颜色属性的主函数。
- **流程**:
  1. 解包 `layer_id`、`color_offset`、`alpha_offset`
  2. 通过 `find_attribute(kg, sd, layer_id)` 查找属性
  3. 如果属性存在：
     - float4/RGBA 类型：读取 float4 值，输出 RGB 到 `color_offset`，Alpha 到 `alpha_offset`
     - float3 类型：读取 float3 值，输出到 `color_offset`，Alpha 固定为 1.0
  4. 如果属性不存在：输出黑色 `(0,0,0)` 和 Alpha `0.0`

### `svm_node_vertex_color_bump_dx`
- **签名**: `ccl_device_noinline void svm_node_vertex_color_bump_dx(KernelGlobals kg, ShaderData *sd, float *stack, const uint4 node)`
- **功能**: 凹凸贴图 x 方向微分偏移版本。
- **差异**: 使用 `dual4`/`dual3` 类型获取属性值及其 x 方向微分，然后计算 `val + dx * bump_filter_width`。`bump_filter_width` 从 `node.z` 以 uint-as-float 方式解码。

### `svm_node_vertex_color_bump_dy`
- **签名**: `ccl_device_noinline void svm_node_vertex_color_bump_dy(KernelGlobals kg, ShaderData *sd, float *stack, const uint4 node)`
- **功能**: 凹凸贴图 y 方向微分偏移版本。逻辑与 `bump_dx` 相同，但使用 `dy` 分量。

## 依赖关系

- **内部头文件**:
  - `kernel/geom/attribute.h` — 属性查找 (`find_attribute`)
  - `kernel/geom/primitive.h` — 图元表面属性插值 (`primitive_surface_attribute`)
  - `kernel/svm/util.h` — SVM 栈操作工具
  - `util/math_base.h` — 基础数学函数
- **被引用**: `kernel/svm/svm.h`

## 实现细节 / 关键算法

1. **属性类型自适应**: 根据属性的实际存储类型（`NODE_ATTR_FLOAT4`/`NODE_ATTR_RGBA` vs `NODE_ATTR_FLOAT3`）选择不同的读取路径。float4 类型可以提供独立的 Alpha 通道，float3 类型的 Alpha 固定为 1.0。

2. **凹凸贴图微分**: bump 变体利用 `primitive_surface_attribute<T>(kg, sd, descriptor, compute_dx, compute_dy)` 的微分参数获取属性值在表面参数空间中的梯度。`bump_filter_width` 控制有限差分步长。

3. **属性缺失处理**: 当指定的颜色属性层不存在时，节点不会报错，而是静默地返回黑色和零 Alpha，保证着色器执行的稳定性。

4. **与通用属性节点的区别**: 相比 `attribute.h` 中的通用属性节点，顶点颜色节点是专门针对颜色属性优化的简化版本。它不支持体积属性和凹凸 P 偏移等通用属性特有的功能，但在颜色+Alpha 的典型用例中更加高效直接。

## 关联文件

- `kernel/svm/attribute.h` — 通用属性节点（功能更全面）
- `kernel/geom/primitive.h` — 图元属性插值底层实现
- `kernel/svm/svm.h` — SVM 主调度器
