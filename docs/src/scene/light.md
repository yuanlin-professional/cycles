# light.h / light.cpp - 光源定义与光源管理器

## 概述

本文件定义了 Cycles 渲染器中的光源系统核心类。`Light` 类继承自 `Geometry`，表示场景中的各种光源类型（点光源、聚光灯、面光源、远距光源、背景光源等）。`LightManager` 负责管理所有光源的设备端数据同步，包括光源分布表、光源树构建、背景光环境贴图采样以及 IES 光域网文件的加载与管理。

## 类与结构体

### Light

- **继承**: `Geometry`（几何体基类）
- **功能**: 描述场景中单个光源的所有属性参数，包含光源类型、强度、尺寸、聚光灯参数、阴影投射、MIS 采样、焦散控制等。
- **关键成员**:
  - `light_type` — 光源类型枚举 (`LightType`)，包括点光源、聚光灯、面光源、远距光、背景光
  - `strength` — 光源颜色强度 (`float3`)
  - `size` — 光源尺寸，影响软阴影范围
  - `sizeu` / `sizev` — 面光源的两个维度尺寸
  - `spread` — 面光源的扩散角度
  - `spot_angle` / `spot_smooth` — 聚光灯锥角及边缘柔化
  - `cast_shadow` — 是否投射阴影
  - `use_mis` — 是否使用多重重要性采样
  - `use_caustics` — 是否计算焦散效果
  - `is_portal` — 是否作为光照入口（Portal Light）
  - `normalize` — 是否按光源表面积归一化功率
  - `max_bounces` — 最大光线反弹次数
  - `map_resolution` — 环境贴图分辨率
  - `average_radiance` — 平均辐射度（用于背景光）
- **关键方法**:
  - `tag_update(Scene*)` — 标记光源已修改，触发场景更新
  - `has_contribution(Scene*, Object*)` — 检查光源是否对场景有贡献
  - `get_shader()` — 获取关联的着色器
  - `area(Transform&)` — 计算变换后的光源面积
  - `compute_bounds()` — 计算包围盒（覆盖自 `Geometry`）
  - `apply_transform()` — 应用变换矩阵（覆盖自 `Geometry`）
  - `primitive_type()` — 返回图元类型（覆盖自 `Geometry`）

### LightManager

- **功能**: 全局光源管理器，负责光源数据的设备端同步、光源分布表构建、光源树生成和背景光贴图采样等。
- **关键成员**:
  - `need_update_background` — 是否需要更新背景光（含 MIS 重要性采样贴图）
  - `ies_slots` — IES 光域网槽位列表，管理 IES 文件的加载和引用计数
  - `ies_mutex` — IES 操作的线程互斥锁
  - `update_flags` — 更新标志位，使用位掩码记录需要更新的内容
- **关键方法**:
  - `device_update(Device*, DeviceScene*, Scene*, Progress&)` — 主更新入口，将光源数据同步到设备
  - `device_free(Device*, DeviceScene*, bool)` — 释放设备端资源
  - `add_ies(string&)` / `add_ies_from_file(string&)` — 加载 IES 光域网数据
  - `remove_ies(int)` — 移除 IES 数据引用
  - `has_background_light(Scene*)` — 检查场景是否有背景光
  - `tag_update(Scene*, uint32_t)` — 通过标志位触发特定更新
  - `need_update()` — 查询是否需要更新

### LightManager 更新标志位

| 标志 | 说明 |
|------|------|
| `MESH_NEED_REBUILD` | 网格需要重建 |
| `EMISSIVE_MESH_MODIFIED` | 发光网格已修改 |
| `LIGHT_MODIFIED` | 光源已修改 |
| `LIGHT_ADDED` | 新增光源 |
| `LIGHT_REMOVED` | 移除光源 |
| `OBJECT_MANAGER` | 对象管理器更新 |
| `SHADER_COMPILED` | 着色器已编译 |
| `SHADER_MODIFIED` | 着色器已修改 |

### IESSlot（内部结构体）

- **功能**: 管理单个 IES 光域网文件的加载数据与引用计数。
- **关键成员**:
  - `ies` — IES 文件解析对象
  - `hash` — 内容哈希值，用于去重
  - `users` — 引用计数

## 核心函数

### 静态辅助函数（light.cpp）

- `shade_background_pixels()` — 在设备上评估背景着色器，生成环境贴图像素数据
- `LightManager::test_enabled_lights()` — 遍历场景光源，禁用无贡献或不支持的光源进行性能优化
- `LightManager::device_update_lights()` — 将所有光源参数打包为 `KernelLight` 数组传输到设备
- `LightManager::device_update_distribution()` — 构建光源采样分布表（CDF），用于按概率采样光源
- `LightManager::device_update_tree()` — 构建光源树（Light Tree），用于高效的多光源采样
- `LightManager::device_update_background()` — 生成背景环境光的 MIS 重要性采样贴图
- `LightManager::device_update_ies()` — 将 IES 光域网数据上传到设备

## 依赖关系

- **内部头文件**:
  - `kernel/types.h` — 内核数据类型定义
  - `graph/node.h` — 节点系统基类
  - `scene/geometry.h` — 几何体基类
  - `util/ies.h` — IES 文件解析
  - `util/thread.h` — 线程工具
  - `scene/background.h`、`scene/film.h`、`scene/integrator.h`、`scene/light_tree.h`、`scene/mesh.h`、`scene/object.h`、`scene/scene.h`、`scene/shader.h`、`scene/shader_graph.h`、`scene/shader_nodes.h`（cpp 中引用）
- **被引用**: 被广泛引用于 `scene/object.cpp`、`scene/scene.cpp`、`scene/integrator.cpp`、`scene/background.cpp`、`scene/light_tree.h`、`scene/light_tree.cpp`、`scene/light_tree_debug.cpp`、`scene/geometry.cpp`、`scene/shader.cpp`、`scene/svm.cpp`、`scene/osl.cpp`、`scene/volume.cpp`、`session/session.cpp`、`hydra/light.cpp`、`app/cycles_xml.cpp` 等约 20 个文件

## 实现细节 / 关键算法

1. **光源启用测试** (`test_enabled_lights`): 遍历所有光源，根据光源类型、着色器状态、MIS 需求等条件判断是否启用。无贡献的光源会被禁用以减少开销。

2. **光源分布构建** (`device_update_distribution`): 为每个光源和发光三角形计算采样权重，构建累积分布函数（CDF）。权重基于光源强度和面积。此分布用于非光源树模式下的光源采样。

3. **背景光 MIS 贴图** (`device_update_background`): 通过在设备上评估背景着色器像素，构建二维 MIS 重要性采样贴图。使用行列边缘分布实现高效的环境光重要性采样。

4. **IES 光域网管理**: 使用引用计数和哈希去重机制管理 IES 文件。多个光源引用同一 IES 文件时共享同一槽位数据。

5. **光源面积计算** (`Light::area`): 根据光源类型计算有效面积——面光源计算矩形/椭圆面积，点光源/聚光灯计算球面积，用于功率归一化。

## 关联文件

- `scene/light_tree.h` / `scene/light_tree.cpp` — 光源树构建与遍历
- `scene/light_tree_debug.h` / `scene/light_tree_debug.cpp` — 光源树调试可视化
- `scene/object.h` / `scene/object.cpp` — 光源作为几何对象被引用
- `scene/background.h` / `scene/background.cpp` — 背景光相关
- `scene/shader.h` — 光源着色器
- `kernel/types.h` — `KernelLight`、`LightType` 等内核数据结构
- `util/ies.h` — IES 文件解析工具
