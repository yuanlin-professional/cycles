# interpolation.h / interpolation.cpp - 细分曲面属性插值

## 概述

本文件实现了细分曲面的属性插值系统，负责将原始网格的顶点属性（UV、颜色、法线等）插值到细分后的三角形网格上。支持线性插值和基于 OpenSubdiv 的平滑插值两种模式，能处理顶点级、角点级、面级及运动模糊属性。该模块是 dice（切割）阶段的核心依赖，在切割过程中通过回调函数实时计算插值属性值。

## 类与结构体

### SubdAttribute

- **功能**: 封装单个细分属性的插值回调及细化数据
- **关键成员**:
  - `interp` — 插值回调函数（`std::function`），签名为 `(patch_index, face_index, corner, vert/triangle_index*, uv*, count)`
  - `refined_data` — OpenSubdiv 细化后的属性数据缓冲区（`vector<char>`）

### SubdAttributeInterpolation

- **功能**: 管理所有需要插值的属性，为 EdgeDice 提供顶点属性和三角形属性的插值列表
- **关键成员**:
  - `mesh` — 网格引用
  - `osd_mesh` / `osd_data` — OpenSubdiv 网格和数据引用（条件编译 `WITH_OPENSUBDIV`）
  - `vertex_attributes` — 顶点级属性插值列表（`vector<SubdAttribute>`）
  - `triangle_attributes` — 三角形级属性插值列表（`vector<SubdAttribute>`）
- **关键方法**:
  - `setup()` — 初始化所有属性插值，遍历 subd 属性并为每个属性创建对应的插值回调
  - `support_interp_attribute(attr)` — 判断属性是否需要插值（排除 PTex 坐标、Catmull-Clark 下的法线等）
  - `setup_attribute(subd_attr, mesh_attr)` — 根据数据类型分派到具体的模板方法
  - `setup_attribute_vertex_linear<T>(...)` — 线性顶点属性插值（四边形双线性/N-gon 中心辅助插值）
  - `setup_attribute_vertex_smooth<T>(...)` — OpenSubdiv 平滑顶点属性插值（使用 PrimvarRefiner 和 PatchTable）
  - `setup_attribute_corner_linear<T>(...)` — 线性角点属性插值
  - `setup_attribute_corner_smooth<T>(...)` — OpenSubdiv 平滑角点属性插值（Face-Varying 通道）
  - `setup_attribute_face<T>(...)` — 面级属性复制到三角形
  - `setup_attribute_type<T>(...)` — 根据属性元素类型分派到相应的插值方法

### SubdFloat<T>

- **功能**: 通用浮点类型的读写适配器模板，用于 float、float2、float3、float4 类型
- **关键成员**: `Type` 和 `AccumType` 均为 `T`

### SubdByte

- **功能**: 字节颜色类型（uchar4）的读写适配器，在插值计算时使用 float4 精度
- **关键方法**:
  - `read(uchar4)` — 转换为 float4 进行计算
  - `output(float4)` — 转换回 uchar4 输出

## 枚举与常量

无独立枚举或常量定义。

## 核心函数

### setup_attribute_vertex_linear<T>()
- **签名**: `void setup_attribute_vertex_linear(const Attribute &subd_attr, Attribute &mesh_attr, const int motion_step = 0)`
- **功能**: 创建线性顶点插值回调。对四边形面使用双线性插值；对 N-gon 面，先计算多边形中心值，然后对当前角的子四边形进行双线性插值。

### setup_attribute_vertex_smooth<T>()
- **签名**: `void setup_attribute_vertex_smooth(const Attribute &subd_attr, Attribute &mesh_attr, const int motion_step = 0)`
- **功能**: （需要 `WITH_OPENSUBDIV`）使用 OpenSubdiv 的 PrimvarRefiner 对属性进行层级细化，然后通过 PatchTable 的 EvaluateBasis 在极限面上求值。同时支持计算运动法线。

### setup_attribute_corner_linear<T>()
- **签名**: `void setup_attribute_corner_linear(const Attribute &subd_attr, Attribute &mesh_attr)`
- **功能**: 创建线性角点插值回调，为每个三角形的3个角点分别计算插值。逻辑与顶点线性插值类似但写入三角形角点数据。

### setup_attribute_corner_smooth<T>()
- **签名**: `void setup_attribute_corner_smooth(Attribute &mesh_attr, const int channel, const vector<char> &merged_values)`
- **功能**: （需要 `WITH_OPENSUBDIV`）使用 Face-Varying 通道的细化数据和 EvaluateBasisFaceVarying 进行平滑角点插值。

### setup_attribute_type<T>()
- **签名**: `void setup_attribute_type(const Attribute &subd_attr, Attribute &mesh_attr)`
- **功能**: 顶层分派函数，根据属性的 element 类型（VERTEX、CORNER、VERTEX_MOTION、FACE）选择合适的插值方法。对 Catmull-Clark 细分类型的特定属性（GENERATED、POSITION_UNDEFORMED 等）使用平滑插值。

## 依赖关系

- **内部头文件**: `subd/osd.h`, `util/types.h`, `util/vector.h`, `scene/attribute.h`, `scene/mesh.h`, `util/color.h`
- **外部库**: `<cstddef>`, `<functional>`；条件依赖 OpenSubdiv（通过 `subd/osd.h`）
- **被引用**: `subd/dice.cpp`, `scene/mesh_subdivision.cpp`

## 实现细节 / 关键算法

1. **延迟插值模式**: 所有插值逻辑被封装为 `std::function` 回调，在 `setup()` 阶段创建。实际插值在 EdgeDice 的 `set_vertex()` 和 `set_triangle()` 中触发，这允许批量并行处理。
2. **N-gon 处理**: 对于非四边形面，先计算多边形所有角的属性值平均作为中心值，再对当前角及其相邻角的中间值组成子四边形进行双线性插值。
3. **字节精度保护**: 对 uchar4 类型属性（如顶点颜色），在整个插值过程中使用 float4 精度，仅在最终写入时转换回字节，避免精度损失。
4. **OpenSubdiv 平滑插值**: 对于 Catmull-Clark 细分类型，使用 PrimvarRefiner 逐层级细化属性值，然后通过 PatchTable 的基函数求值在极限面上获得精确的插值结果。

## 关联文件

- `subd/osd.h` / `subd/osd.cpp` — OpenSubdiv 封装，提供细化器和面片表
- `subd/dice.h` / `subd/dice.cpp` — 切割器，消费插值回调
- `scene/attribute.h` — 属性数据结构定义
- `scene/mesh.h` — 网格数据结构，提供面和角信息
- `scene/mesh_subdivision.cpp` — 网格细分主流程
