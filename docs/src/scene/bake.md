# bake.h / bake.cpp - 纹理烘焙管理器

## 概述

本文件定义了 Cycles 渲染器的纹理烘焙管理器。`BakeManager` 类负责管理烘焙模式的启停状态，控制烘焙过程中是否使用摄像机和随机种子，并将烘焙目标对象信息同步到设备端内核。烘焙模式允许将路径追踪的渲染结果（如光照、法线等）烘焙到纹理贴图上。

## 类与结构体

### BakeManager

- **继承**: 无（独立类）
- **功能**: 管理纹理烘焙的全局状态和设备端数据同步。
- **关键成员**:
  - `need_update_` — 是否需要更新（私有）
  - `use_baking_` — 是否启用烘焙模式（私有）
  - `use_camera_` — 烘焙是否考虑摄像机方向（私有）
  - `use_seed_` — 烘焙是否使用随机种子通道（私有）
- **关键方法**:
  - `set_baking(Scene*, bool)` — 启用或禁用烘焙模式。状态变更时自动标记 Film 和 Integrator 需要更新
  - `get_baking()` — 查询当前烘焙状态
  - `set_use_camera(bool)` — 设置是否使用摄像机（影响烘焙方向）
  - `set_use_seed(bool)` / `get_use_seed()` — 设置/查询是否使用烘焙种子通道
  - `device_update(Device*, DeviceScene*, Scene*, Progress&)` — 将烘焙参数同步到设备。扫描场景对象找到标记为烘焙目标的网格，记录其对象索引和三角形偏移量
  - `device_free(Device*, DeviceScene*)` — 释放设备资源（当前为空操作）
  - `tag_update()` — 手动标记需要更新
  - `need_update()` — 查询是否需要更新

## 核心函数

所有逻辑封装在 `BakeManager` 类方法中。

## 依赖关系

- **内部头文件**:
  - `device/device.h` — 设备接口
  - `scene/scene.h` — 场景类
  - `util/progress.h` — 进度报告
  - `scene/integrator.h`、`scene/mesh.h`、`scene/object.h`、`scene/shader.h`、`scene/stats.h`（cpp 中引用）
  - `session/buffers.h`（cpp 中引用）
- **被引用**: `scene/film.cpp`、`scene/integrator.cpp`、`scene/scene.cpp`、`session/tile.cpp`、`scene/bake.cpp`

## 实现细节 / 关键算法

1. **烘焙模式激活** (`set_baking`): 当烘焙状态发生变化时，标记 `Film` 为已修改并触发 `Integrator` 的全量更新 (`UPDATE_ALL`)，因为烘焙模式会影响渲染通道配置和积分器行为（如过滤透明闭包）。

2. **设备更新** (`device_update`): 核心逻辑是在启用烘焙模式时遍历场景中的所有对象，找到标记为烘焙目标（`get_is_bake_target()` 返回真）且为网格类型的对象。将该对象的索引（`object_index`）和几何体的图元偏移量（`prim_offset`）写入 `KernelBake` 结构体，供内核在烘焙采样时定位目标三角形。

3. **摄像机方向**: `use_camera` 标志控制烘焙时是否从摄像机方向发射射线。设为 false 时从表面法线方向采样，适用于大多数烘焙场景；设为 true 时从摄像机方向采样，适用于需要视角依赖的效果。

4. **烘焙种子通道**: 当 `use_seed` 为真时，`Film::update_passes()` 会自动添加 `PASS_BAKE_SEED` 通道，允许每个像素使用不同的随机种子以减少采样模式。

## 关联文件

- `scene/film.h` / `scene/film.cpp` — 烘焙模式影响通道配置（添加 `PASS_BAKE_PRIMITIVE`、`PASS_BAKE_DIFFERENTIAL`、`PASS_BAKE_SEED`）
- `scene/integrator.h` / `scene/integrator.cpp` — 烘焙模式影响积分器配置（过滤透明闭包）
- `scene/object.h` — `Object::get_is_bake_target()` 和 `Object::index`
- `scene/geometry.h` — `Geometry::prim_offset` 图元偏移量
- `kernel/types.h` — `KernelBake` 设备端数据结构
