# film.h / film.cpp - 胶片/成像平面参数与渲染通道管理

## 概述

本文件定义了 Cycles 渲染器的胶片（成像平面）系统。`Film` 类继承自 `Node`，管理渲染输出的所有参数，包括曝光度、像素滤波器、雾效参数、Cryptomatte、阴影捕获器等。该类的核心职责是管理渲染通道（Pass）的自动创建、去重、排序以及设备端偏移量映射，确保内核能正确写入和读取各类渲染通道数据。

## 类与结构体

### FilterType（枚举）

| 类型 | 说明 |
|------|------|
| `FILTER_BOX` | 盒形滤波器 |
| `FILTER_GAUSSIAN` | 高斯滤波器 |
| `FILTER_BLACKMAN_HARRIS` | Blackman-Harris 窗函数滤波器 |

### Film

- **继承**: `Node`（节点系统基类）
- **功能**: 管理胶片参数和渲染通道系统，控制像素滤波和各类渲染输出通道的配置。
- **关键成员**:
  - `exposure` — 曝光度
  - `pass_alpha_threshold` — Alpha 阈值，低于此值的像素不写入某些通道
  - `display_pass` — 视口显示的通道类型
  - `show_active_pixels` — 是否显示活跃像素
  - `filter_type` — 像素滤波类型
  - `filter_width` — 滤波器宽度
  - `mist_start` / `mist_depth` / `mist_falloff` — 雾效通道参数
  - `cryptomatte_passes` — Cryptomatte 通道类型（物体/材质/资产）
  - `cryptomatte_depth` — Cryptomatte 深度
  - `use_approximate_shadow_catcher` — 是否使用近似阴影捕获器
  - `use_sample_count` — 是否使用采样计数通道
  - `filter_table_offset_` — 滤波器查找表在设备端的偏移量（私有）
- **关键方法**:
  - `add_default(Scene*)` — 添加默认的 Combined 通道
  - `device_update(Device*, DeviceScene*, Scene*)` — 计算所有通道偏移量、构建滤波器表、同步到设备
  - `device_free(Device*, DeviceScene*, Scene*)` — 释放滤波器查找表
  - `get_aov_offset(Scene*, string, bool&)` — 获取自定义 AOV 通道的偏移量
  - `update_lightgroups(Scene*)` — 更新光组映射，返回是否发生变化
  - `update_passes(Scene*)` — 核心方法：根据场景配置自动创建/更新所有必需的渲染通道
  - `get_kernel_features(Scene*)` — 获取所需的内核特性标志（降噪、光照通道、AO 通道等）
  - `add_auto_pass()` — 自动添加渲染通道（私有）
  - `remove_auto_passes()` — 移除所有自动生成的通道（私有）
  - `finalize_passes()` — 去重、排序并最终确定通道列表（私有）

## 核心函数

### 像素滤波器函数（静态）

- `filter_func_box()` — 盒形滤波，返回常数 1.0
- `filter_func_gaussian()` — 高斯滤波，`exp(-2 * (6v/width)^2)`
- `filter_func_blackman_harris()` — Blackman-Harris 窗函数滤波，四项余弦和
- `filter_table()` — 基于选定的滤波函数生成逆 CDF 重要性采样查找表

### 通道排序

- `compare_pass_order()` — 通道排序比较函数：首先按分量数降序，同分量数内非光组通道优先，最后按类型升序。此排序保证 AOV 和 Cryptomatte 通道在内核中按预期顺序排列。

## 依赖关系

- **内部头文件**:
  - `scene/pass.h` — 渲染通道 `Pass` 类定义
  - `kernel/types.h` — `KernelFilm`、`PassType` 等内核类型
  - `graph/node.h` — 节点系统基类
  - `scene/background.h`、`scene/bake.h`、`scene/camera.h`、`scene/integrator.h`、`scene/mesh.h`、`scene/object.h`、`scene/scene.h`、`scene/tables.h`（cpp 中引用）
- **被引用**: `scene/scene.h`、`scene/scene.cpp`、`scene/integrator.cpp`、`scene/light.cpp`、`session/tile.cpp`、`integrator/path_trace_work.cpp`、`app/cycles_xml.cpp` 等约 9 个文件

## 实现细节 / 关键算法

1. **渲染通道自动管理** (`update_passes`): 这是胶片系统最复杂的方法。它根据场景配置自动创建必需的渲染通道：
   - 始终创建 Combined 通道
   - 自适应采样时创建 `PASS_SAMPLE_COUNT` 和 `PASS_ADAPTIVE_AUX_BUFFER`
   - 降噪时按需创建法线和反照率通道
   - 阴影捕获器需要 `PASS_SHADOW_CATCHER`、`PASS_SHADOW_CATCHER_SAMPLE_COUNT`、`PASS_SHADOW_CATCHER_MATTE`
   - 烘焙模式添加 `PASS_BAKE_PRIMITIVE`、`PASS_BAKE_DIFFERENTIAL`、`PASS_BAKE_SEED`
   - 体积场景添加体积散射/透射通道及其降噪版本、主成分通道
   - 为光照通道自动添加所需的分量通道（直接/间接/除法通道）

2. **通道偏移量映射** (`device_update`): 遍历所有通道，为每个通道计算在像素缓冲区中的偏移量（`pass_stride`），将偏移量写入 `KernelFilm` 结构体的对应字段。未使用的通道标记为 `PASS_UNUSED` 以跳过内核中的掩码测试。

3. **通道去重与排序** (`finalize_passes`): 合并相同类型和模式的重复通道（保留非空名称），使用稳定排序按分量数和类型排列，确保 Cryptomatte 等依赖顺序的通道正确对齐。

4. **像素滤波器查找表**: 使用逆 CDF 方法构建重要性采样表（`FILTER_TABLE_SIZE` 个条目），高斯滤波器的宽度乘以 3 倍、Blackman-Harris 乘以 2 倍以覆盖有效范围。

5. **光组管理**: 为每个具有光组标签的通道建立从光组名称到索引的映射，检测映射变化以触发场景更新。

## 关联文件

- `scene/pass.h` — `Pass` 类和 `PassInfo` 定义
- `scene/integrator.h` — 自适应采样和降噪配置
- `scene/background.h` — 透明背景影响阴影捕获器
- `scene/bake.h` — 烘焙模式通道
- `scene/tables.h` — 滤波器查找表管理
- `kernel/types.h` — `KernelFilm` 设备端数据结构
