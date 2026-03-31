# background.h / background.cpp - 背景/环境光设置

## 概述

本文件定义了 Cycles 渲染器的背景系统。`Background` 类继承自 `Node`，管理场景背景的着色器、可见性、透明度以及光组等参数。背景既可以作为环境光照来源（通过背景着色器），也可以配置为透明以用于合成工作流。该类负责将这些配置同步到设备端的 `KernelBackground` 结构体。

## 类与结构体

### Background

- **继承**: `Node`（节点系统基类）
- **功能**: 管理场景背景的渲染属性，包括着色器选择、射线可见性过滤、透明度设置和光组分配。
- **关键成员**:
  - `use_shader` — 是否使用自定义背景着色器（否则使用默认空背景）
  - `shader` — 背景着色器指针（`Shader*`）
  - `visibility` — 射线可见性位掩码，控制背景对不同射线类型的可见性（漫射、光泽、透射、散射、摄像机）
  - `transparent` — 是否使用透明背景
  - `transparent_glass` — 是否将玻璃材质视为透明
  - `transparent_roughness_threshold` — 透明玻璃的粗糙度阈值
  - `lightgroup` — 所属光组名称 (`ustring`)
- **关键方法**:
  - `device_update(Device*, DeviceScene*, Scene*)` — 将背景参数同步到设备内核
  - `device_free(Device*, DeviceScene*)` — 释放设备资源（当前为空操作）
  - `tag_update(Scene*)` — 当背景着色器被修改时触发更新标记
  - `get_shader(Scene*)` — 获取实际使用的背景着色器：若 `use_shader` 为真，返回指定着色器或默认背景着色器；否则返回空着色器

## 核心函数

无额外的自由函数。所有逻辑封装在 `Background` 类的方法中。

## 依赖关系

- **内部头文件**:
  - `graph/node.h` — 节点系统基类
  - `util/types.h` — 基础类型
  - `scene/integrator.h`、`scene/light.h`、`scene/scene.h`、`scene/shader.h`、`scene/shader_graph.h`、`scene/shader_nodes.h`（cpp 中引用）
  - `device/device.h`（cpp 中引用）
- **被引用**: `scene/light.cpp`、`scene/film.cpp`、`scene/integrator.cpp`、`scene/scene.cpp`、`scene/shader.cpp`、`scene/svm.cpp`、`scene/osl.cpp`、`scene/volume.cpp`、`session/session.cpp`、`session/tile.cpp`、`hydra/session.cpp`、`app/cycles_xml.cpp` 等约 13 个文件

## 实现细节 / 关键算法

1. **着色器可见性控制** (`device_update`): 将 `visibility` 位掩码转换为内核着色器排除标志。当背景着色器图仅有一个节点（即无实际节点连接）时，设置 `SHADER_EXCLUDE_ANY` 以完全跳过背景评估。否则根据各射线类型可见性位逐一设置排除标志：
   - `SHADER_EXCLUDE_DIFFUSE` — 对漫射射线不可见
   - `SHADER_EXCLUDE_GLOSSY` — 对光泽射线不可见
   - `SHADER_EXCLUDE_TRANSMIT` — 对透射射线不可见
   - `SHADER_EXCLUDE_SCATTER` — 对体积散射射线不可见
   - `SHADER_EXCLUDE_CAMERA` — 对摄像机射线不可见

2. **透明玻璃处理**: 当同时启用 `transparent` 和 `transparent_glass` 时，将粗糙度阈值平方两次存储——一次用于 Principled BSDF 约定，一次用于内核中与各向异性粗糙度的快速比较。禁用时设为 -1.0 以跳过测试。

3. **体积着色器**: 若背景着色器包含体积属性（`has_volume`），则将背景的体积着色器索引设为与表面着色器相同；否则设为 `SHADER_NONE`。

4. **光组分配**: 从场景的光组映射表中查找背景所属光组的索引。未找到时设为 `LIGHTGROUP_NONE`。

5. **智能更新标记** (`tag_update`): 当背景着色器被修改时，仅标记 `use_shader` 属性为已修改，而非标记整个对象，以避免不必要的更新开销。

## 关联文件

- `scene/shader.h` / `scene/shader.cpp` — 背景着色器定义
- `scene/light.h` / `scene/light.cpp` — 背景光源处理（MIS 采样贴图）
- `scene/film.h` / `scene/film.cpp` — 透明背景影响阴影捕获器通道
- `scene/scene.h` — `default_background` 和 `default_empty` 着色器
- `kernel/types.h` — `KernelBackground` 设备端数据结构
