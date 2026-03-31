# pointcloud.h / pointcloud.cpp - Hydra点云渲染图元（Rprim）

## 概述

本文件实现了 Cycles 的 Hydra 点云渲染图元（Rprim），将通用场景描述（USD）中的 `HdPoints` 数据转换为 Cycles 内部的 `PointCloud` 几何体。点云图元以球体形式渲染每个点，支持逐点位置、半径及自定义 Primvar 属性。

点云常用于粒子系统、散布效果以及点扫描数据的可视化渲染。

## 类与结构体

### HdCyclesPoints

- **继承**: `HdCyclesGeometry<PXR_NS::HdPoints, CCL_NS::PointCloud>`
- **功能**: 实现点云数据的同步，将 Hydra 点坐标和宽度转换为 Cycles 点云几何体
- **关键方法**:
  - `Populate()` -- 核心同步方法，处理点位置、宽度和 Primvar 的填充，并为每个点分配着色器索引
  - `PopulatePoints()` -- 从场景委托获取点坐标（支持 ExtComputation），转换为 `float3`
  - `PopulateWidths()` -- 读取宽度数据，支持 Constant 和 Vertex 插值，转换为半径（宽度 / 2）
  - `PopulatePrimvars()` -- 遍历 Vertex 和 Constant 插值的 Primvar 描述符并写入属性
  - `GetInitialDirtyBitsMask()` -- 返回初始脏标志（Points/Widths/Primvar）
  - `_PropagateDirtyBits()` -- Points 和 Widths 始终联动更新（两者互相依赖）

## 核心函数

无独立核心函数。

## 依赖关系

- **内部头文件**: `hydra/pointcloud.h`, `hydra/attribute.h`, `hydra/geometry.inl`, `scene/pointcloud.h`
- **外部依赖**: `pxr/imaging/hd/points.h`, `pxr/imaging/hd/extComputationUtils.h`
- **被引用**: `render_delegate.cpp`

## 实现细节 / 关键算法

1. **点和宽度联动**: `_PropagateDirtyBits()` 确保 `DirtyPoints` 和 `DirtyWidths` 始终同时标记，因为点云的点数量变化时半径数组也必须重新生成。

2. **着色器数组**: `Populate()` 在更新点和宽度后，为每个点创建着色器索引数组（全部为 0，即使用默认材质），通过 `_geom->set_shader()` 设置。

3. **重建判定**: 当点数量发生变化时（`_geom->num_points() != numPoints`）标记 `rebuild = true`，触发 BVH 重建。

4. **宽度处理**: 与曲线图元类似，Hydra 的 `widths` 表示直径，转换为 Cycles 的半径需乘以 0.5f。Constant 插值时将单一值扩展到所有点。

5. **Primvar 插值映射**:
   - Vertex -> `ATTR_ELEMENT_VERTEX`（逐点）
   - Constant -> `ATTR_ELEMENT_OBJECT`（逐对象）
   - 支持 `textureCoordinate` 角色映射到 `ATTR_STD_UV`
   - 支持 `displayColor` / `color` 角色映射到 `ATTR_STD_VERTEX_COLOR`
   - 支持 `normals` 映射到 `ATTR_STD_VERTEX_NORMAL`

## 关联文件

- `hydra/geometry.h` / `hydra/geometry.inl` -- 基类模板 `HdCyclesGeometry`
- `hydra/attribute.h` -- `ApplyPrimvars()` 属性写入辅助
- `hydra/render_delegate.h` -- 通过 `CreateRprim()` 创建本类实例
- `scene/pointcloud.h` -- Cycles 内部 PointCloud 几何体定义
