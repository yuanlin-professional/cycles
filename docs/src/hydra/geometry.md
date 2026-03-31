# geometry.h / geometry.cpp / geometry.inl - Hydra几何体基类模板

## 概述

本文件定义并实现了 Cycles Hydra渲染代理 中所有几何渲染图元（Rprim）的通用基类模板 `HdCyclesGeometry`。该模板封装了几何体的初始化、场景同步、实例管理、材质绑定、变换更新和生命周期管理等公共逻辑。

`geometry.h` 声明模板接口，`geometry.inl` 提供完整的模板实现，`geometry.cpp` 是为使 clangd 等工具正常工作的哑文件。所有具体几何类型（Mesh、Hair、PointCloud、Volume）均继承此模板。

## 类与结构体

### HdCyclesGeometry\<Base, CyclesBase\>

- **继承**: 模板参数 `Base`（如 `HdMesh`、`HdBasisCurves`、`HdPoints`、`HdVolume`）
- **模板参数**:
  - `Base` -- Hydra 基类（`HdMesh`、`HdBasisCurves`、`HdPoints`、`HdVolume`）
  - `CyclesBase` -- Cycles 几何体类型（`Mesh`、`Hair`、`PointCloud`、`Volume`）
- **功能**: 提供几何图元的通用同步框架，管理 Cycles 几何体对象和实例对象的创建、更新与销毁
- **关键成员**:
  - `_geom` (`CyclesBase*`) -- Cycles 几何体对象指针（由 `Initialize()` 创建）
  - `_instances` (`vector<Object*>`) -- Cycles 实例对象列表（至少包含一个默认实例）
  - `_geomTransform` (`GfMatrix4d`) -- 几何体自身变换矩阵
- **关键方法**:
  - `Sync()` -- Hydra 同步入口，协调初始化、实例化器同步、材质绑定、变换更新和子类 `Populate()` 调用
  - `Initialize()` -- 延迟初始化，首次同步时在场景中创建几何体节点和默认实例
  - `InitializeInstance()` -- 初始化单个实例：绑定几何体、设置 instanceId 属性、默认颜色和随机 ID
  - `Finalize()` -- 清理几何体和所有实例，从场景中删除节点（除非 `keep_nodes` 为 true）
  - `Populate()` -- 纯虚函数，由子类实现具体的几何数据填充逻辑
  - `GetInitialDirtyBitsMask()` -- 返回基础脏标志（PrimID/Transform/MaterialId/Visibility/Instancer）
  - `_PropagateDirtyBits()` -- 默认直接返回（子类可覆盖以实现脏标志传播）
  - `_InitRepr()` -- 空实现（Cycles 不使用 Hydra 的表示模式系统）
  - `GetPrimvarInterpolation()` -- 辅助方法，查询指定 Primvar 的插值模式

## 核心函数

- `convert_transform()` -- 外部声明（定义在 `mesh.cpp`），将 `GfMatrix4d` 转换为 Cycles `Transform`

## 依赖关系

- **内部头文件**:
  - `hydra/geometry.h` -- 模板声明
  - `hydra/instancer.h`, `hydra/material.h`, `hydra/session.h` -- 在 `geometry.inl` 中使用
  - `scene/geometry.h`, `scene/object.h`, `scene/scene.h`, `util/hash.h` -- Cycles 核心依赖
- **外部依赖**: `pxr/imaging/hd/rprim.h`, `pxr/imaging/hd/sceneDelegate.h`
- **被引用**: `mesh.h`, `curves.h`, `pointcloud.h`, `volume.h`（通过 `geometry.h`）；`mesh.cpp`, `curves.cpp`, `pointcloud.cpp`, `volume.cpp`（通过 `geometry.inl`）

## 实现细节 / 关键算法

1. **Sync 流程（geometry.inl）**:
   - 检查是否为 Clean 状态，若是则直接返回
   - 调用 `Initialize()` 确保几何体和默认实例已创建
   - 同步实例化器（`_UpdateInstancer` + `_SyncInstancerAndParents`，PXR >= 2102）
   - 更新可见性
   - 获取场景锁（`SceneLock`）
   - 材质绑定：从渲染索引获取 `HdCyclesMaterial`，设置 `used_shaders`；无材质时回退到 `default_surface`
   - 更新 PrimID（加 1 偏移，后续在 AOV 中修正）
   - 更新变换矩阵（含场景单位缩放 `metersPerUnit`）
   - 处理实例化：通过 `HdCyclesInstancer::ComputeInstanceTransforms()` 获取变换矩阵数组，动态调整实例数量
   - 更新可见性标志
   - 调用子类 `Populate()` 填充几何数据
   - 标记几何体和实例的更新
   - 清除脏标志

2. **实例管理**:
   - 非实例化时：单个默认实例，instanceId 属性设为 -1.0f
   - 实例化时：根据变换矩阵数量动态增删实例对象，instanceId 设为实例索引（float）
   - 实例化器的第一个实例的 instanceId 属性设为 +0.0f（表示首个实例）

3. **延迟初始化**: `Initialize()` 通过检查 `_geom == nullptr` 实现幂等性，首次调用时创建几何体节点（`scene->create_node<CyclesBase>()`）和默认实例对象

4. **Finalize 的 keep_nodes 模式**: 当 `HdCyclesSession::keep_nodes` 为 true 时（外部传入的 Session），不删除几何体和实例节点，仅清除本地引用

5. **变换计算**: 最终变换 = `scale(metersPerUnit)` * `convert_transform(geomTransform * instanceTransform)`，其中 `metersPerUnit` 处理场景单位与 Cycles 内部单位的转换

## 关联文件

- `hydra/mesh.h`, `hydra/curves.h`, `hydra/pointcloud.h`, `hydra/volume.h` -- 具体几何类型子类
- `hydra/instancer.h` -- 实例化变换计算
- `hydra/material.h` -- 材质着色器绑定
- `hydra/session.h` -- 场景锁和会话管理
- `hydra/config.h` -- 命名空间定义
