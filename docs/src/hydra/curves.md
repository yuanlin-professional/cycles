# curves.h / curves.cpp - Hydra曲线（毛发）渲染图元（Rprim）

## 概述

本文件实现了 Cycles 的 Hydra 曲线渲染图元（Rprim），将通用场景描述（USD）中的 `HdBasisCurves` 数据转换为 Cycles 内部的 `Hair` 几何体。主要用于毛发、草地、睫毛等基于样条曲线的几何表示。

曲线图元支持控制点位置、宽度（半径）以及自定义 Primvar 属性的同步。

## 类与结构体

### HdCyclesCurves

- **继承**: `HdCyclesGeometry<PXR_NS::HdBasisCurves, CCL_NS::Hair>`
- **功能**: 实现基础曲线的同步，将 Hydra 的 basisCurves 拓扑和属性映射到 Cycles Hair 几何体
- **关键方法**:
  - `Populate()` -- 核心同步方法，依次处理拓扑、控制点、宽度和 Primvar
  - `PopulateTopology()` -- 清空几何体后，按曲线段数和控制点总数预留空间，逐条曲线调用 `add_curve()`
  - `PopulatePoints()` -- 从场景委托获取控制点数据（支持 ExtComputation），转换为 `float3` 并设置 `curve_keys`
  - `PopulateWidths()` -- 读取宽度数据，支持 Constant（全局统一半径）和 Vertex（逐控制点半径）插值，宽度除以 2 转为半径
  - `PopulatePrimvars()` -- 遍历所有 Primvar 描述符，映射到 Cycles 属性元素
  - `GetInitialDirtyBitsMask()` -- 返回初始脏标志（Points/Widths/Primvar/Topology）
  - `_PropagateDirtyBits()` -- 拓扑变更时触发控制点、宽度和 Primvar 的全量重建

## 核心函数

无独立核心函数，所有逻辑封装在类方法中。

## 依赖关系

- **内部头文件**: `hydra/curves.h`, `hydra/attribute.h`, `hydra/geometry.inl`, `scene/hair.h`
- **外部依赖**: `pxr/imaging/hd/basisCurves.h`, `pxr/imaging/hd/extComputationUtils.h`
- **被引用**: `render_delegate.cpp`

## 实现细节 / 关键算法

1. **拓扑构建**: 通过 `GetBasisCurvesTopology()` 获取曲线顶点计数数组，使用 `reserve_curves()` 预分配内存，然后逐条曲线调用 `add_curve(key, 0)`，其中 `key` 为当前曲线起始控制点的索引，`0` 表示使用第一个（默认）材质着色器。

2. **宽度到半径转换**: Hydra 的 `widths` 表示直径，Cycles 使用半径，因此所有宽度值乘以 0.5f。Constant 插值时将单一值扩展到所有控制点。

3. **Primvar 插值映射**:
   - Vertex/Varying -> `ATTR_ELEMENT_CURVE_KEY`（逐控制点）
   - Uniform -> `ATTR_ELEMENT_CURVE`（逐曲线段）
   - Constant -> `ATTR_ELEMENT_OBJECT`（逐对象）

4. **重建判定**: `rebuild` 标志由 `curve_keys_is_modified()` 或 `curve_radius_is_modified()` 决定，用于通知场景是否需要重建 BVH。

5. **displayColor 处理**: 当 Constant 插值的 `displayColor` Primvar 存在时，将其设置为对象颜色（`_instances[0]->set_color()`）。

## 关联文件

- `hydra/geometry.h` / `hydra/geometry.inl` -- 基类模板 `HdCyclesGeometry`
- `hydra/attribute.h` -- `ApplyPrimvars()` 属性写入辅助
- `hydra/render_delegate.h` -- 通过 `CreateRprim()` 创建本类实例
- `scene/hair.h` -- Cycles 内部 Hair 几何体定义
