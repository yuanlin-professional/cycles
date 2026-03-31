# attribute.h - 属性节点的SVM实现

## 概述

`attribute.h` 实现了着色器虚拟机(SVM)中的属性(Attribute)节点。该节点用于读取几何体上的自定义属性数据（如 UV 坐标、顶点组、自定义数据层等），支持 float、float2、float3、float4/RGBA 等多种数据类型，并能处理表面属性和体积属性。此外还提供了凹凸贴图所需的微分偏移变体(bump_dx/bump_dy)。

## 核心函数

### `svm_node_attr_init`
- **签名**: `ccl_device AttributeDescriptor svm_node_attr_init(KernelGlobals kg, ShaderData *sd, const uint4 node, NodeAttributeOutputType *type, uint *out_offset)`
- **功能**: 初始化属性查找。从节点数据解包输出偏移和类型信息，然后在对象的属性表中查找指定属性。如果属性未找到，返回一个默认描述符。

### `svm_node_attr_store`（4个重载版本）
- **签名**: 接受 `float`、`float2`、`float3`、`float4` 类型的值
- **功能**: 根据请求的输出类型（`NODE_ATTR_OUTPUT_FLOAT`、`NODE_ATTR_OUTPUT_FLOAT3`、`NODE_ATTR_OUTPUT_FLOAT_ALPHA`）将属性值存储到 SVM 栈中。
  - float -> float3 时扩展为 `(f, f, f)`
  - float3 -> float 时取平均值 `average(f)`
  - float4 的 alpha 通道通过 `f.w` 获取

### `svm_surface_attr<T>` / `svm_surface_attr_dx<T>` / `svm_surface_attr_dy<T>`
- **功能**: 模板函数，分别读取表面属性值、x方向微分偏移值、y方向微分偏移值。微分版本用于凹凸贴图(bump mapping)计算。

### `svm_node_attr`
- **签名**: `template<uint node_feature_mask> void svm_node_attr(KernelGlobals kg, ShaderData *sd, float *stack, const uint4 node)`
- **功能**: 主属性节点处理函数。按优先级处理：
  1. **体积属性**: 如果是体积图元，使用 `volume_attribute_float4` 读取体积属性（支持随机采样）
  2. **灯光 UV**: 对于灯光图元的 UV 属性，返回 `(1-u-v, u, 0)` 的重心坐标
  3. **Generated 坐标回退**: 如果 `ATTR_STD_GENERATED` 属性不存在，回退到物体空间坐标
  4. **表面属性**: 根据属性数据类型调用对应的 `svm_surface_attr<T>`

### `svm_node_attr_bump_dx` / `svm_node_attr_bump_dy`
- **功能**: 属性节点的凹凸微分变体。在 x 或 y 方向上对位置进行微分偏移后读取属性值，用于 bump mapping 的有限差分法计算梯度。

### `svm_node_bump_P_dx` / `svm_node_bump_P_dy`
- **功能**: 辅助函数，计算在 x/y 方向偏移后的位置 `P + dPdx * bump_filter_width`。

## 依赖关系

- **内部头文件**:
  - `kernel/globals.h` — 全局内核数据
  - `kernel/geom/attribute.h` — 属性查找 (`find_attribute`)
  - `kernel/geom/object.h` — 物体变换（逆位置变换）
  - `kernel/geom/primitive.h` — 图元表面属性插值
  - `kernel/geom/volume.h` — 体积属性读取
  - `kernel/svm/util.h` — SVM 栈操作工具
  - `kernel/util/differential.h` — 微分类型定义
- **被引用**: `kernel/svm/svm.h`、`kernel/svm/tex_coord.h`、`kernel/svm/geometry.h`

## 实现细节 / 关键算法

1. **属性类型分发**: 属性以不同精度存储（`NODE_ATTR_FLOAT`、`NODE_ATTR_FLOAT2`、`NODE_ATTR_FLOAT3`、`NODE_ATTR_FLOAT4`/`NODE_ATTR_RGBA`），读取时使用模板特化根据存储类型调用不同的 `primitive_surface_attribute<T>` 函数。

2. **输出类型转换**: `svm_node_attr_store` 的重载系统实现了灵活的类型转换。例如，float4 属性在 `NODE_ATTR_OUTPUT_FLOAT` 模式下输出 RGB 平均值，在 `NODE_ATTR_OUTPUT_FLOAT_ALPHA` 模式下输出 alpha 通道。

3. **凹凸贴图微分**: `bump_dx/dy` 变体使用 `dual<T>` 类型获取属性值及其微分，然后按 `val + dx * bump_filter_width` 计算偏移值。这种有限差分方法用于在着色器中估算法线扰动的梯度。

4. **体积随机采样**: 体积属性读取支持 `stochastic_sample` 参数（来自 `node.w`），用于体素数据的随机插值。

## 关联文件

- `kernel/geom/attribute.h` — 属性查找底层实现
- `kernel/geom/primitive.h` — 图元表面属性插值
- `kernel/svm/svm.h` — SVM 主调度器
- `kernel/svm/tex_coord.h` — 纹理坐标节点（也使用属性查找）
- `kernel/svm/geometry.h` — 几何节点（也使用属性查找）
