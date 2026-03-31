# mesh.h / mesh.cpp - Hydra多边形网格渲染图元（Rprim）

## 概述

本文件实现了 Cycles 的 Hydra 多边形网格渲染图元（Rprim），将通用场景描述（USD）中的 `HdMesh` 数据（顶点、法线、拓扑、Primvar 属性）转换为 Cycles 内部的 `Mesh` 几何体。支持三角化网格和 Catmull-Clark / 线性细分曲面两种模式。

该类是几何数据从 Hydra 场景同步到 Cycles 渲染器的关键桥梁之一。

## 类与结构体

### HdCyclesMesh

- **继承**: `HdCyclesGeometry<PXR_NS::HdMesh, CCL_NS::Mesh>`
- **功能**: 实现多边形网格的同步、拓扑三角化/细分、法线处理和 Primvar 属性映射
- **关键成员**:
  - `_util` (`HdMeshUtil`) -- USD 网格工具类，用于三角化计算
  - `_topology` (`HdMeshTopology`) -- 缓存的网格拓扑数据
  - `_primitiveParams` (`VtIntArray`) -- 三角化后的面参数映射表（三角面索引到原始面索引）
- **关键方法**:
  - `Populate()` -- 核心同步方法，按顺序执行拓扑、顶点、法线、Primvar 的填充
  - `PopulateTopology()` -- 解析拓扑，执行三角化或设置细分参数（含折痕/角点权重）
  - `PopulatePoints()` -- 读取顶点位置（支持 ExtComputation 计算结果），转换为 Cycles `float3` 格式
  - `PopulateNormals()` -- 处理多种插值模式的法线数据（常量/均匀/顶点/面变化）
  - `PopulatePrimvars()` -- 遍历所有 Primvar 描述符，进行三角化重映射后写入 Cycles 属性集
  - `Finalize()` -- 清理拓扑缓存，调用基类清理逻辑
  - `GetInitialDirtyBitsMask()` -- 返回初始脏标志（Points/Normals/Primvar/Topology/DisplayStyle/SubdivTags）
  - `_PropagateDirtyBits()` -- 脏标志传播：材质变更触发拓扑更新，拓扑变更触发全量重建

## 核心函数

- `convert_transform()` -- 将 `GfMatrix4d` 转换为 Cycles 的 `Transform` 结构体（列主序到行主序）
- `ComputeTriangulatedUniformPrimvar<T>()` -- 模板函数，将均匀插值（每面一值）的 Primvar 按三角化映射展开
- `ComputeTriangulatedFaceVaryingPrimvar()` -- 将面变化插值的 Primvar 通过 `HdMeshUtil` 进行三角化重映射

## 依赖关系

- **内部头文件**: `hydra/mesh.h`, `hydra/attribute.h`, `hydra/geometry.inl`, `scene/mesh.h`
- **外部依赖**: `pxr/imaging/hd/mesh.h`, `pxr/imaging/hd/meshUtil.h`, `pxr/imaging/hd/extComputationUtils.h`
- **被引用**: `render_delegate.cpp`

## 实现细节 / 关键算法

1. **拓扑处理策略**:
   - 无细分时：使用 `HdMeshUtil::ComputeTriangleIndices()` 将多边形三角化，生成 `_primitiveParams` 映射表
   - 细分时：直接以原始面数据设置 `subd_face`，并处理 `SubdivTags` 中的折痕边、折痕顶点权重

2. **几何子集材质**: 通过 `HdGeomSubsets` 为不同面集分配不同材质着色器索引，构建 `faceShaders` 查找表

3. **法线处理**:
   - 仅非细分网格处理显式法线
   - 支持 Constant（全局统一）、Uniform（按面）、Vertex/Varying（按顶点）、FaceVarying（按角）插值
   - FaceVarying 法线当前标记为 TODO（Cycles 尚不支持逐角法线）

4. **Primvar 映射**: 遍历五种插值模式，将 Hydra Primvar 映射到 Cycles 的 `AttributeElement`：
   - FaceVarying -> ATTR_ELEMENT_CORNER
   - Uniform -> ATTR_ELEMENT_FACE
   - Vertex/Varying -> ATTR_ELEMENT_VERTEX
   - Constant -> ATTR_ELEMENT_OBJECT
   特殊处理 `displayColor`（Constant 时设为对象颜色）和 `textureCoordinate`（映射到 ATTR_STD_UV）

5. **脏标志传播**: 材质变更会强制拓扑更新（因为几何子集的着色器需重新绑定）；拓扑变更会清空几何体并触发全量重填充。

## 关联文件

- `hydra/geometry.h` / `hydra/geometry.inl` -- 基类模板 `HdCyclesGeometry`
- `hydra/attribute.h` -- `ApplyPrimvars()` 属性写入辅助
- `hydra/material.h` -- 材质着色器查询（通过 `HdCyclesMaterial`）
- `hydra/render_delegate.h` -- 通过 `CreateRprim()` 创建本类实例
