# services.h + services.cpp - 开放着色语言(OSL)渲染服务接口与实现

## 概述

`services.h` 和 `services.cpp` 实现了 `OSLRenderServices` 类,这是开放着色语言(OSL)与 Cycles 渲染器之间的核心桥梁。该类继承自 `OSL::RendererServices`,为 OSL 着色器执行过程中所需的矩阵变换、属性查询、纹理查找、点云访问和光线追踪等功能提供了 Cycles 特定的实现。此外,还管理了纹理句柄映射和闭包注册等渲染会话级别的资源。

## 类与结构体

### `OSLTextureHandle`
纹理句柄结构体,扩展了 OSL 的默认纹理句柄以支持 Cycles 的多种纹理来源:

| 成员 | 类型 | 说明 |
|------|------|------|
| `type` | `Type` 枚举 | 纹理类型: `OIIO`(OIIO 纹理系统)、`SVM`(着色器虚拟机(SVM)纹理槽)、`IES`(IES 光度文件)、`BEVEL`(倒角 AO)、`AO`(环境光遮蔽) |
| `svm_slots` | `vector<int4>` | 压缩的 tile-svm_slot 映射对,格式: x:tile_a, y:slot_a, z:tile_b, w:slot_b |
| `oiio_handle` | `TextureHandle*` | OIIO 纹理系统句柄 |
| `processor` | `ColorSpaceProcessor*` | 色彩空间转换处理器 |
| `handle` | `ImageHandle` | Cycles 图像句柄 |

### `OSLTextureHandleMap`
- 类型别名: `OIIO::unordered_map_concurrent<OSLUStringHash, OSLTextureHandle>`
- 线程安全的并发哈希表,存储纹理文件名到句柄的映射。支持 OSL 编译器并行编译着色器时的并发查找和插入。

### `OSLRenderServices`
继承自 `OSL::RendererServices` 的渲染服务类:

**公开成员**:
| 成员 | 类型 | 说明 |
|------|------|------|
| `textures` | `OSLTextureHandleMap` | 纹理句柄并发哈希表 |
| `image_manager` | `ImageManager*` (static) | 图像管理器指针 |

**私有成员**:
| 成员 | 类型 | 说明 |
|------|------|------|
| `device_type_` | `int` | 设备类型(用于 OptiX 支持判断) |

**静态 ustring 常量** (约 60+ 个):
定义了所有 OSL 属性查询中使用的字符串常量,涵盖:
- 坐标空间: `u_world`, `u_camera`, `u_screen`, `u_raster`, `u_ndc`
- 对象属性: `u_object_location`, `u_object_color`, `u_object_index`, `u_object_random` 等
- 几何属性: `u_geom_name`, `u_geom_trianglevertices`, `u_geom_undisplaced`, `u_is_smooth` 等
- 粒子属性: `u_particle_index`, `u_particle_age`, `u_particle_location` 等
- 毛发属性: `u_is_curve`, `u_curve_thickness`, `u_curve_length`, `u_curve_random` 等
- 点属性: `u_is_point`, `u_point_radius`, `u_point_position` 等
- 路径属性: `u_path_ray_length`, `u_path_ray_depth`, `u_path_diffuse_depth` 等
- 光线追踪: `u_trace`, `u_hit`, `u_hitdist`, `u_N`, `u_Ng`, `u_P`, `u_I`, `u_u`, `u_v`
- 特殊纹理: `u_at_bevel`, `u_at_ao`
- 相机属性: `u_sensor_size`, `u_image_resolution`, `u_aperture_size`, `u_focal_distance` 等

## 核心函数

### 构造与析构

#### `OSLRenderServices(OSL::TextureSystem *texture_system, int device_type)`
- 初始化渲染服务,传入纹理系统和设备类型。

#### `~OSLRenderServices()`
- 析构时输出 OIIO 纹理系统统计信息到日志。

### 功能支持查询

#### `supports(string_view feature) const`
- 返回对特定功能的支持状态。当前仅在 OptiX 设备上返回对 "OptiX" 的支持。

### 矩阵变换 (8 个重载)

#### `get_matrix` / `get_inverse_matrix` 系列
提供四组重载,分别支持:
1. **TransformationPtr + time**: 获取对象的局部到世界矩阵(支持运动模糊)
2. **ustring(hash) + time**: 获取命名坐标空间的变换矩阵(world, camera, screen, raster, NDC)
3. **TransformationPtr (无 time)**: 获取对象矩阵(不考虑运动模糊)
4. **ustring(hash) (无 time)**: 获取命名坐标空间矩阵

实现逻辑:
- 对象空间和着色器空间均映射到对象变换矩阵(`object_get_transform` / `object_get_inverse_transform`)
- 支持运动模糊时,通过 `object_fetch_transform_motion_test` 获取指定时间的插值变换
- 命名空间映射到 `kernel_data.cam` 中预计算的相机矩阵

### 属性查询

#### `get_attribute(sg, derivatives, object, type, name, val)`
- **功能**: 获取着色点上的属性值。依次尝试以下查找策略:
  1. 用户数据(`get_userdata`)
  2. 对象标准属性(位置、颜色、索引等)
  3. 粒子属性
  4. 毛发/曲线属性
  5. 点云属性
  6. 自定义几何属性

#### `get_array_attribute(sg, derivatives, object, type, name, index, val)`
- **功能**: 获取数组属性的指定索引元素。

#### `get_userdata(derivatives, name, type, sg, val)`
- **功能**: 获取用户数据(几何属性)。通过属性描述符查找和 `primitive_surface_attribute` 读取实际值。

#### 静态属性查询函数:
- `get_background_attribute` - 获取背景着色器属性(支持路径深度查询)
- `get_camera_attribute` - 获取相机属性(传感器大小、分辨率、光圈等)
- `get_object_standard_attribute` - 获取对象标准属性(位置、颜色、索引、几何信息等)

### 纹理查找

#### `get_texture_handle(filename, context, options)`
- **功能**: 获取或创建纹理句柄。根据文件名判断纹理类型:
  - `@bevel`: 创建 BEVEL 类型句柄
  - `@ao`: 创建 AO 类型句柄
  - 普通文件名: 查找或创建 OIIO 纹理句柄,处理色彩空间转换

#### `texture(filename, handle, ..., result, ...)`
- **功能**: 2D 纹理查找。根据句柄类型分发:
  - `SVM`: 通过 Cycles 内部纹理系统查找
  - `IES`: 查找 IES 光度曲线
  - `OIIO`: 通过 OIIO TextureSystem 查找
  - `AO`/`BEVEL`: 计算环境光遮蔽或倒角法线

#### `texture3d(filename, handle, ..., P, ..., result, ...)`
- **功能**: 3D 纹理查找。对 SVM 纹理使用三线性插值(`kernel_tex_image_interp_3d`),对 OIIO 纹理调用纹理系统的 3D 查找。

#### `environment(filename, handle, ..., R, ..., result, ...)`
- **功能**: 环境贴图查找。通过 OIIO 的环境纹理查找接口实现。

#### `get_texture_info(filename, handle, ..., dataname, datatype, data, ...)`
- **功能**: 查询纹理元数据(如分辨率、数据类型等)。

### 光线追踪

#### `trace(options, sg, P, dPdx, dPdy, R, dRdx, dRdy)`
- **功能**: OSL `trace()` 函数的实现。构造光线并调用 Cycles 内核的 BVH 场景求交。结果缓存在 `OSLTraceData` 中,延迟计算着色数据。

#### `getmessage(sg, source, name, type, val, derivatives)`
- **功能**: 获取 `trace()` 调用结果中的消息数据。支持查询:
  - `hit`: 是否命中
  - `hitdist`: 命中距离
  - `N`, `Ng`: 着色法线和几何法线
  - `P`: 命中位置
  - `I`: 入射方向
  - `u`, `v`: UV 坐标

### 闭包注册

#### `register_closures(OSL::ShadingSystem *ss)` (static)
- **功能**: 向 OSL ShadingSystem 注册所有 Cycles 支持的闭包类型。通过包含 `closures_template.h` 自动注册所有标准闭包,并手动注册 `layer` 闭包。

### 点云接口

#### `pointcloud_search` / `pointcloud_get` / `pointcloud_write`
- **功能**: 点云查找接口。当前实现为空(返回 0/false),Cycles 不支持点云。

## 依赖关系

### services.h
- **内部头文件**:
  - `<OSL/oslclosure.h>`, `<OSL/oslexec.h>`, `<OSL/rendererservices.h>` - OSL 渲染服务基类
  - `<OpenImageIO/unordered_map_concurrent.h>` - 并发哈希表
  - `scene/image.h` - 图像句柄定义
  - `kernel/osl/compat.h` - OSL 版本兼容层
  - `kernel/osl/types.h` - OSL 类型定义
- **被引用**:
  - `src/kernel/osl/closures.cpp` - 调用 `register_closures`
  - `src/kernel/osl/services.cpp` - 自身实现文件

### services.cpp
- **内部头文件**:
  - `scene/colorspace.h`, `scene/object.h` - 场景层数据
  - `kernel/device/cpu/image.h` - CPU 端图像查找
  - `kernel/osl/globals.h`, `kernel/osl/services.h`, `kernel/osl/services_shared.h` - OSL 组件
  - `kernel/integrator/state.h` - 积分器状态
  - `kernel/geom/primitive.h`, `kernel/geom/shader_data.h` - 几何与着色数据
  - `kernel/bvh/bvh.h` - BVH 加速结构(trace 功能)
  - `kernel/camera/camera.h` - 相机功能
  - `kernel/svm/ao.h`, `kernel/svm/bevel.h` - 着色器虚拟机(SVM)的 AO 和倒角实现
  - `kernel/util/ies.h`, `kernel/util/texture_3d.h` - IES 和 3D 纹理工具

## 实现细节 / 关键算法

### 矩阵转换

`copy_matrix` 辅助函数将 Cycles 的 `Transform`(3x4 行优先)或 `ProjectionTransform`(4x4 行优先)转换为 OSL 的 `Matrix44`(4x4 列优先),通过转置实现行列顺序的转换。

### 纹理句柄缓存

纹理句柄通过 `OIIO::unordered_map_concurrent` 缓存,该并发哈希表支持无锁读取和有锁写入,适合 OSL 编译器并行编译着色器的场景。纹理句柄和纹理句柄映射表存储在 `OSLRenderServices` 而非 `OSLGlobals` 中,因为它们需要在多个渲染会话之间共享,而纹理句柄是着色系统的一部分。

### trace() 的延迟求值

`trace()` 函数仅执行 BVH 求交,不立即计算着色数据(ShaderData)。只有当后续通过 `getmessage()` 查询需要着色数据的属性(如法线、UV 等)时,才调用 `shader_setup_from_ray` 进行延迟初始化。

### 色彩空间处理

纹理查找后,若关联了 `ColorSpaceProcessor`,会自动对查找结果进行色彩空间转换。对于 SVM 纹理,由于已在加载时处理了色彩空间,不需要额外转换。

### 属性查询优先级

属性查询按以下顺序执行,命中即返回:
1. 内置几何属性(bump_map_normal、trianglevertices 等)
2. 对象标准属性(location、color、index 等)
3. 粒子系统属性
4. 曲线/毛发属性
5. 点云属性
6. 自定义用户属性(通过属性描述符查找)

## 关联文件

- `src/kernel/osl/globals.h` - OSL 全局数据结构
- `src/kernel/osl/closures.cpp` - 调用 `register_closures` 注册闭包
- `src/kernel/osl/services_gpu.h` - GPU 端渲染服务(OptiX)
- `src/kernel/osl/services_shared.h` - CPU/GPU 共享的服务函数
- `src/scene/osl.cpp` - 场景层 OSL 集成,创建和配置 `OSLRenderServices`
- `src/kernel/svm/ao.h` - AO 纹理的着色器虚拟机(SVM)实现
- `src/kernel/svm/bevel.h` - 倒角纹理的着色器虚拟机(SVM)实现
