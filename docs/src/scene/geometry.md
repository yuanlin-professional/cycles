# geometry.h / geometry.cpp — 几何体基类与几何管理器

## 概述

`geometry.h` 和 `geometry.cpp` 定义了 Cycles 渲染器中所有几何类型（网格、毛发、体积、点云）的抽象基类 `Geometry`，以及负责设备数据同步的 `GeometryManager`。`Geometry` 继承自 `Node`，提供属性、变换、运动模糊、层次包围体(BVH) 构建等通用接口。`GeometryManager` 统筹所有几何体的设备更新流程，包括曲面细分、位移、BVH 构建与属性上传。

额外的实现分布在 `geometry_attributes.cpp`（属性上传至设备）、`geometry_bvh.cpp`（BVH 构建与打包）和 `geometry_mesh.cpp`（网格/曲线/点云数据打包上传）中。

## 类与结构体

### Geometry

- **继承**: `Node`
- **功能**: 所有几何类型的抽象基类，定义通用几何操作接口
- **关键成员**:
  - `geometry_type` (`Type`) — 几何类型枚举：`MESH`、`HAIR`、`VOLUME`、`POINTCLOUD`、`LIGHT`
  - `attributes` (`AttributeSet`) — 几何体上的属性集合
  - `used_shaders` (`array<Node*>`) — 引用的着色器列表
  - `bounds` (`BoundBox`) — 包围盒
  - `transform_applied` (`bool`) — 变换是否已应用到顶点
  - `transform_negative_scaled` (`bool`) — 是否存在负缩放
  - `transform_normal` (`Transform`) — 法线变换矩阵
  - `motion_steps` (`uint`) — 运动模糊步数
  - `use_motion_blur` (`bool`) — 是否启用运动模糊
  - `bvh` (`unique_ptr<BVH>`) — 几何体自有的 BVH
  - `attr_map_offset` (`size_t`) — 属性映射在设备数组中的偏移
  - `prim_offset` (`size_t`) — 图元在全局数组中的偏移
  - `has_volume` (`bool`) — 是否包含体积着色器
  - `has_surface_bssrdf` (`bool`) — 是否包含次表面散射
  - `need_update_rebuild` (`bool`) — 是否需要完整重建
- **关键方法**:
  - `clear()` — 清除几何数据
  - `compute_bounds()` — 纯虚函数，计算包围盒
  - `apply_transform()` — 纯虚函数，应用变换到几何数据
  - `need_attribute()` — 检查场景是否需要某个标准属性或命名属性
  - `needed_attributes()` — 收集所有引用着色器请求的属性集
  - `get_uv_tiles()` — 纯虚函数，获取 UDIM 瓦片列表
  - `compute_bvh()` — 构建几何体的 BVH
  - `need_build_bvh()` — 判断是否需要独立 BVH（实例化、OptiX/Metal 布局等）
  - `is_instanced()` — 判断几何体是否被多对象引用
  - `has_true_displacement()` — 是否有真实位移着色器
  - `has_motion_blur()` — 是否有运动模糊
  - `tag_update()` — 标记需要更新
  - `is_mesh()` / `is_hair()` / `is_pointcloud()` / `is_volume()` / `is_light()` — 类型检查辅助函数

### GeometryManager

- **功能**: 管理所有几何体的设备同步、BVH 构建、曲面细分和位移
- **关键成员**:
  - `update_flags` (`uint32_t`) — 更新标志位集合
  - `need_flags_update` (`bool`) — 是否需要更新标志
- **关键方法**:
  - `device_update_preprocess()` — 预处理：检测变更、创建体积网格、设置设备数组标签
  - `device_update()` — 主更新流程：法线计算 -> 曲面细分 -> 位移 -> 属性上传 -> BVH 构建 -> 网格打包
  - `device_free()` — 释放设备内存
  - `tag_update()` — 标记更新并联动 `ObjectManager`
  - `collect_statistics()` — 收集渲染统计信息
  - `displace()` — 在设备上执行位移着色器求值（定义在 `mesh_displace.cpp`）
  - `create_volume_mesh()` — 从体积数据创建代理网格（定义在 `volume.cpp`）
  - `geom_calc_offset()` — 计算各几何体在全局数组中的偏移
  - `device_update_mesh()` — 打包网格/曲线/点云数据到设备（定义在 `geometry_mesh.cpp`）
  - `device_update_attributes()` — 打包属性数据到设备（定义在 `geometry_attributes.cpp`）
  - `device_update_bvh()` — 构建场景级 BVH（定义在 `geometry_bvh.cpp`）

### 设备更新标志枚举

- `DEVICE_CURVE_DATA_MODIFIED` / `DEVICE_MESH_DATA_MODIFIED` / `DEVICE_POINT_DATA_MODIFIED` — 数据已修改
- `ATTR_FLOAT_MODIFIED` ~ `ATTR_UCHAR4_MODIFIED` — 各类型属性已修改
- `*_NEED_REALLOC` — 各数据需要重新分配

## 核心函数

- `Geometry::compute_bvh()` — 根据设备参数构建几何体 BVH，支持 Embree、OptiX、Metal、HIPRT 等多种布局
- `GeometryManager::device_update()` — 完整的设备更新流水线：
  1. 法线计算
  2. 曲面细分（OpenSubdiv Catmull-Clark / 线性）
  3. 切线计算
  4. 位移图像加载
  5. 属性上传
  6. 位移着色器求值
  7. 静态变换应用
  8. BVH 构建（对象级 + 场景级）
  9. 网格数据打包上传

## 依赖关系

- **内部头文件**: `graph/node.h`, `bvh/params.h`, `scene/attribute.h`, `util/boundbox.h`, `util/set.h`, `util/transform.h`, `util/types.h`, `util/vector.h`
- **被引用**: `scene/mesh.h`, `scene/hair.h`, `scene/pointcloud.h`, `scene/object.h`, `scene/light.h`, `scene/volume.cpp`, `scene/geometry_attributes.cpp`, `scene/geometry_bvh.cpp`, `scene/geometry_mesh.cpp` 等

## 实现细节 / 关键算法

- **偏移量计算** (`geom_calc_offset`): 遍历所有几何体，按类型累计计算顶点、三角形、曲线段、点的全局偏移量。偏移变化时在 OptiX/Metal 静态 BVH 布局下会触发重建。
- **增量更新**: 通过精细的标志位系统（`MODIFIED` vs `NEED_REALLOC`）区分仅数据修改和需要重新分配的情况，减少不必要的内存操作。
- **并行处理**: 曲面细分和 BVH 构建使用 `TaskPool` 和 `parallel_for_each` 进行并行化。
- **实例判断**: `is_instanced()` 考虑了变换是否已应用以及是否有次表面散射（BSSRDF 需要独立 BVH）。

## 关联文件

- `geometry_attributes.cpp` — `update_svm_attributes()` 属性映射上传
- `geometry_bvh.cpp` — `Geometry::compute_bvh()` 和 `device_update_bvh()` BVH 构建
- `geometry_mesh.cpp` — `device_update_mesh()` 网格/曲线/点云数据打包
- `scene/mesh.h` / `scene/hair.h` / `scene/pointcloud.h` / `scene/volume.h` — 具体几何子类
- `scene/object.h` — 场景对象，引用几何体
- `scene/attribute.h` — 属性系统
