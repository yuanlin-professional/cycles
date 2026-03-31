# scene.h / scene.cpp - 场景管理核心

## 概述

本文件定义了 Cycles 渲染器的核心场景类 `Scene`，是整个渲染数据的中央容器和管理者。它持有所有渲染对象（相机、灯光、几何体、着色器、渲染通道等）以及对应的管理器（ImageManager、LightManager、ShaderManager 等），负责协调场景数据到设备端的同步更新流程。场景更新按照严格的依赖顺序执行，确保各管理器间的数据一致性。

## 类与结构体

### SceneParams
- **功能**: 场景全局参数配置
- **关键成员**:
  - `shadingsystem` — 着色系统类型（SVM 或 OSL）
  - `bvh_layout` — 请求的 BVH 布局（AUTO、BVH2、BVH8 等）
  - `bvh_type` — BVH 类型（STATIC 或 DYNAMIC）
  - `use_bvh_spatial_split` — 是否启用 BVH 空间分割
  - `use_bvh_compact_structure` — 是否使用紧凑 BVH 结构
  - `use_bvh_unaligned_nodes` — 是否使用非对齐 BVH 节点
  - `num_bvh_time_steps` — BVH 运动模糊时间步数
  - `hair_subdivisions` — 头发细分级别
  - `hair_shape` — 头发形状（RIBBON 或 THICK）
  - `texture_limit` — 纹理尺寸上限（0 表示无限制）
  - `background` — 是否为后台渲染
- **关键方法**:
  - `modified()` — 检查参数是否与另一组参数不同
  - `curve_subdivisions()` — 计算实际曲线细分数（匹配 Embree 限制，1-16）

### Scene
- **继承**: `NodeOwner`
- **功能**: 渲染场景的中央容器，管理所有渲染数据和子系统
- **核心数据成员**:
  - `name` — 场景名称（用于日志）
  - `lightgroups` — 灯光组名称到渲染通道 ID 的映射
  - `bvh` — BVH 加速结构
  - `lookup_tables` — 查找表管理器
  - `camera`, `dicing_camera` — 渲染相机和细分相机
  - `film` — 胶片/成像平面设置
  - `background` — 背景设置
  - `integrator` — 积分器设置
- **数据列表**:
  - `shaders`, `passes`, `geometry`, `objects`, `procedurals`, `particle_systems`, `cameras`, `films`, `integrators`, `backgrounds` — 各类节点的 `unique_ptr_vector` 容器
- **管理器**:
  - `image_manager` — 图像管理器
  - `light_manager` — 灯光管理器
  - `shader_manager` — 着色器管理器
  - `geometry_manager` — 几何体管理器
  - `object_manager` — 物体管理器
  - `osl_manager` — OSL 管理器
  - `particle_system_manager` — 粒子系统管理器
  - `bake_manager` — 烘焙管理器
  - `procedural_manager` — 程序化管理器
  - `volume_manager` — 体积管理器
- **默认着色器**: `default_surface`, `default_volume`, `default_light`, `default_background`, `default_empty`
- **设备相关**: `device` — 渲染设备, `dscene` — 设备端场景数据
- **关键方法**:
  - `device_update()` — 核心更新流程，按依赖顺序将场景数据同步到设备
  - `update()` — 检查是否需要更新并执行 `device_update()`
  - `need_update()` / `need_reset()` / `need_data_update()` — 各级更新检查
  - `reset()` — 重置场景，标记所有管理器需要更新
  - `device_free()` — 释放设备端内存
  - `collect_statistics()` — 收集渲染统计（几何体 + 图像）
  - `create_node<T>()` — 模板工厂方法，创建节点并注册到对应数组，标记管理器更新
  - `delete_node<T>()` — 模板删除方法，从数组移除节点并标记管理器更新
  - `delete_nodes<T>()` — 批量删除同类节点
  - `need_motion()` — 检查是否需要运动模糊或运动向量渲染通道
  - `has_shadow_catcher()` — 检查是否存在阴影捕捉器
  - `has_volume()` — 检查是否存在体积物体
  - `load_kernels()` — 加载渲染内核
  - `update_kernel_features()` — 计算并更新内核特性标志

## 核心函数

- `log_kernel_features()` — 内部辅助函数，打印当前请求的内核特性标志
- `assert_same_owner()` — 调试断言辅助，验证节点集合的所有者一致性

## 依赖关系

- **内部头文件**: `bvh/params.h`, `scene/devicescene.h`, `scene/film.h`, `scene/image.h`, `scene/shader.h`, `util/param.h`, `util/string.h`, `util/thread.h`, `util/unique_ptr.h`, `util/unique_ptr_vector.h`
- **cpp 额外引用**: 大量场景子模块（`background.h`, `bake.h`, `camera.h`, `curves.h`, `hair.h`, `integrator.h`, `light.h`, `mesh.h`, `object.h`, `osl.h`, `particles.h`, `pointcloud.h`, `procedural.h`, `shader.h`, `svm.h`, `tables.h`, `volume.h`）, `session/session.h`, `util/guarded_allocator.h`, `util/log.h`, `util/progress.h`
- **被引用**: 几乎所有 `scene/` 目录下的文件，以及 `session/`, `integrator/`, `hydra/`, `bvh/`, `app/` 等模块（共 46 个文件）

## 实现细节 / 关键算法

1. **更新顺序**: `device_update()` 严格按以下依赖顺序更新：灯光组 -> 着色器编译 -> 渲染通道 -> 内核特性 -> 内核加载 -> 着色器上传 -> 程序化生成 -> 背景 -> 相机 -> 几何预处理 -> 物体 -> 粒子 -> 网格/几何 -> 物体标志 -> 图元偏移 -> 图像 -> 体积 -> 相机体积 -> 查找表 -> 灯光 -> 积分器 -> 胶片 -> 烘焙 -> 常量内存
2. **内核热加载**: 在 `device_update()` 循环中，若检测到内核特性变化，会释放互斥锁后重新加载内核。如果加载期间场景被修改（`scene_updated_while_loading_kernels`），则重新执行着色器编译等前期步骤。
3. **工厂模式**: `create_node<T>()` 使用模板特化为每种节点类型（Light、Mesh、Object、Shader 等 13 种）提供自动注册和管理器标记。
4. **体积栈大小**: `get_volume_stack_size()` 通过统计体积物体间的相交关系估算渲染所需的体积栈深度，最小为 2（背景 + 终止符），最大为 `MAX_VOLUME_STACK_SIZE`。
5. **内核特性收集**: `update_kernel_features()` 遍历所有物体，汇总所需内核特性（毛发类型、运动模糊、阴影捕捉、焦散/MNEE、光链接等），并从着色器、胶片、积分器和相机获取额外特性。
6. **闭包数上限**: `get_max_closure_count()` 统计所有引用着色器的最大闭包数，OSL 模式下固定使用 `MAX_CLOSURE`。
7. **释放顺序**: `free_memory()` 严格按依赖关系反序释放：程序化 -> 物体 -> 几何 -> 粒子 -> 渲染通道 -> 相机/胶片/背景设备释放 -> 着色器 -> 各管理器设备释放。

## 关联文件

- `scene/devicescene.h/.cpp` — 设备端场景数据容器
- `scene/film.h/.cpp` — 胶片/成像平面管理
- `scene/image.h/.cpp` — 图像管理
- `scene/stats.h/.cpp` — 渲染统计
- `scene/procedural.h/.cpp` — 程序化生成
- `session/session.cpp` — 渲染会话（持有并驱动 Scene）
