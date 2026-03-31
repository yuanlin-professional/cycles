# volume.h / volume.cpp - Hydra体积渲染图元（Rprim）

## 概述

本文件实现了 Cycles 的 Hydra 体积渲染图元（Rprim），将通用场景描述（USD）中的 `HdVolume` 数据转换为 Cycles 内部的 `Volume` 几何体。通过关联的 OpenVDB 场资产（Field），将体积密度、颜色、火焰、温度、速度等属性加载到渲染场景中。

体积图元依赖 OpenVDB 支持（`WITH_OPENVDB` 编译宏），是烟雾、火焰、云雾等体积效果的渲染基础。

## 类与结构体

### HdCyclesVolume

- **继承**: `HdCyclesGeometry<PXR_NS::HdVolume, CCL_NS::Volume>`
- **功能**: 实现体积数据的同步，从 Hydra 场描述符加载 OpenVDB 网格数据到 Cycles 体积属性
- **关键方法**:
  - `Populate()` -- 核心同步方法，当体积场脏标志触发时遍历所有场描述符并加载数据
  - `GetInitialDirtyBitsMask()` -- 在基类脏标志基础上添加 `DirtyVolumeField`

## 核心函数

无独立核心函数。

## 依赖关系

- **内部头文件**: `hydra/volume.h`, `hydra/field.h`, `hydra/geometry.inl`, `scene/volume.h`
- **外部依赖**: `pxr/imaging/hd/volume.h`
- **被引用**: `render_delegate.cpp`

## 实现细节 / 关键算法

1. **场描述符遍历**: 通过 `sceneDelegate->GetVolumeFieldDescriptors()` 获取体积关联的所有场描述符（`HdVolumeFieldDescriptor`），每个描述符包含 `fieldId`（指向 Bprim）和 `fieldName`（属性名称）。

2. **OpenVDB 资产查找**: 使用场描述符的 `fieldId` 在渲染索引中查找 `openvdbAsset` 类型的 Bprim（`HdCyclesField`），获取其 `ImageHandle`。

3. **标准属性映射**: 将场名称与 Cycles 标准属性名称匹配：
   - `density` -> `ATTR_STD_VOLUME_DENSITY`
   - `color` -> `ATTR_STD_VOLUME_COLOR`
   - `flame` -> `ATTR_STD_VOLUME_FLAME`
   - `heat` -> `ATTR_STD_VOLUME_HEAT`
   - `temperature` -> `ATTR_STD_VOLUME_TEMPERATURE`
   - `velocity` -> `ATTR_STD_VOLUME_VELOCITY`
   - 非标准名称作为自定义 float 类型的体素属性添加

4. **按需加载**: 仅当场景中的着色器实际需要某属性时（`need_attribute()`），才会创建 `Attribute` 并关联体素数据。

5. **网格合并**: 所有场数据加载完毕后调用 `_geom->merge_grids(scene)` 合并体积网格，`rebuild` 始终标记为 `true`。

## 关联文件

- `hydra/geometry.h` / `hydra/geometry.inl` -- 基类模板 `HdCyclesGeometry`
- `hydra/field.h` -- `HdCyclesField`，OpenVDB 场资产的 Bprim 封装
- `hydra/render_delegate.h` -- 通过 `CreateRprim()` 创建本类实例（需 `WITH_OPENVDB`）
- `scene/volume.h` -- Cycles 内部 Volume 几何体定义
