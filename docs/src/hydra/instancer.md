# instancer.h / instancer.cpp - Hydra实例化器（Instancer）

## 概述

本文件实现了 Cycles 的 Hydra 实例化器，负责管理渲染图元（Rprim）的实例化变换。`HdCyclesInstancer` 从场景委托获取平移、旋转、缩放和自定义变换矩阵等实例 Primvar，并计算每个实例的最终变换矩阵。

实例化器是实现大量重复几何体（如森林中的树木、建筑中的窗户等）高效渲染的核心组件，支持嵌套实例化（父子 Instancer 层级）。

## 类与结构体

### HdCyclesInstancer

- **继承**: `PXR_NS::HdInstancer`
- **功能**: 同步实例化 Primvar 数据，计算原型几何体的实例变换矩阵
- **关键成员**:
  - `_translate` (`VtVec3fArray`) -- 实例平移向量数组
  - `_rotate` (`VtVec4fArray`) -- 实例旋转四元数数组（w, x, y, z）
  - `_scale` (`VtVec3fArray`) -- 实例缩放向量数组
  - `_instanceTransform` (`VtMatrix4dArray`) -- 实例自定义变换矩阵数组
- **关键方法**:
  - `Sync()` -- 同步实例化器状态（PXR_VERSION > 2011），调用 `_UpdateInstancer()` 和 `SyncPrimvars()`
  - `SyncPrimvars()` -- 从场景委托读取实例化 Primvar（translate/rotate/scale/instanceTransform）
  - `ComputeInstanceTransforms()` -- 计算指定原型的所有实例最终变换矩阵，支持嵌套实例化

## 核心函数

无独立核心函数。

## 依赖关系

- **内部头文件**: `hydra/instancer.h`, `hydra/config.h`
- **外部依赖**: `pxr/imaging/hd/instancer.h`, `pxr/base/gf/matrix4d.h`, `pxr/base/gf/quatd.h`, `pxr/imaging/hd/sceneDelegate.h`
- **被引用**: `render_delegate.cpp`, `geometry.inl`

## 实现细节 / 关键算法

1. **Primvar 同步**: `SyncPrimvars()` 遍历 `HdInterpolationInstance` 级别的 Primvar 描述符，根据名称识别并存储 translate、rotate、scale、instanceTransform 四类数据。支持两套 Token 命名：
   - PXR_VERSION < 2311: `translate`, `rotate`, `scale`, `instanceTransform`
   - PXR_VERSION >= 2311: `instanceTranslations`, `instanceRotations`, `instanceScales`, `instanceTransforms`

2. **变换矩阵计算**: `ComputeInstanceTransforms()` 的流程如下：
   - 从场景委托获取实例索引列表和实例化器全局变换
   - 对每个实例，依次应用：全局变换 -> 平移 -> 旋转 -> 缩放 -> 自定义矩阵
   - 旋转使用 `GfQuatd` 四元数构造旋转矩阵

3. **嵌套实例化**: 如果当前实例化器有父实例化器（`GetParentId()` 非空），递归调用父实例化器的 `ComputeInstanceTransforms()`，将父变换与本地变换做矩阵乘法，实现多层级实例化。

4. **版本兼容**: 对 PXR_VERSION <= 2011，`SyncPrimvars()` 在 `ComputeInstanceTransforms()` 中被延迟调用（因为旧版 API 没有 `Sync()` 回调）。

5. **脏状态清理**: `SyncPrimvars()` 结束后调用 `MarkInstancerClean()` 清除脏标志。

## 关联文件

- `hydra/geometry.inl` -- `HdCyclesGeometry::Sync()` 中调用 `ComputeInstanceTransforms()` 获取变换矩阵
- `hydra/render_delegate.h` -- 通过 `CreateInstancer()` 创建本类实例
- `hydra/config.h` -- 命名空间和前向声明
