# pointcloud.h / pointcloud.cpp — 点云几何体

## 概述

`pointcloud.h` 和 `pointcloud.cpp` 实现了 Cycles 的点云几何类 `PointCloud`，继承自 `Geometry`。点云将每个点表示为一个带半径的球体，支持运动模糊和 UDIM 贴图。数据结构简洁高效，每个点仅需坐标、半径和着色器索引。

## 类与结构体

### PointCloud

- **继承**: `Geometry`
- **功能**: 点云几何体，管理点坐标、半径和着色器数据
- **关键成员**:
  - `points` (`array<float3>`) — 点坐标数组
  - `radius` (`array<float>`) — 每个点的半径
  - `shader` (`array<int>`) — 每个点的着色器索引
- **关键方法**:
  - `resize()` / `reserve()` — 调整/预分配点数据
  - `add_point()` — 添加一个点（坐标、半径、着色器索引）
  - `copy_center_to_motion_step()` — 将当前点坐标+半径复制到运动模糊步属性
  - `compute_bounds()` — 计算包围盒（考虑半径和运动步）
  - `apply_transform()` — 应用变换（坐标变换，半径按均匀缩放因子缩放）
  - `get_uv_tiles()` — 获取 UDIM 瓦片
  - `pack()` — 将数据打包为 `float4`（xyz=坐标, w=半径）和着色器 ID 数组
  - `primitive_type()` — 返回 `PRIMITIVE_POINT` 或 `PRIMITIVE_MOTION_POINT`
  - `num_points()` / `num_attributes()` — 数量查询
  - `get_point()` — 获取第 i 个点的辅助结构

### PointCloud::Point

- **功能**: 单个点的辅助结构
- **关键成员**: `index` — 点在数组中的索引
- **关键方法**:
  - `bounds_grow()` — 多个重载，以球体方式扩展包围盒
  - `motion_key()` — 在运动步之间线性插值（坐标+半径）
  - `point_for_step()` — 获取特定运动步的点数据

## 核心函数

### 数据打包

`PointCloud::pack()` 将点坐标和半径打包为 `float4` 数组，着色器索引通过 `ShaderManager::get_shader_id()` 转换为内核着色器 ID。使用缓存的 `last_shader` 避免重复查找。

### 变换应用

`apply_transform()` 与毛发类似，使用变换矩阵三列向量的行列式的立方根作为均匀缩放因子来缩放半径。运动步的数据（`float4` 格式）也进行相应变换。

### 包围盒计算

遍历所有点，使用 `bnds.grow(point, radius)` 以球体方式扩展包围盒。对运动步数据同样处理。如果检测到无效坐标（NaN/Inf），回退到 `grow_safe` 模式。

## 依赖关系

- **内部头文件**: `scene/geometry.h`
- **被引用**: `scene/object.cpp`, `scene/geometry.cpp`, `scene/geometry_mesh.cpp`, `scene/attribute.cpp`, `scene/scene.cpp`

## 实现细节 / 关键算法

- **运动模糊存储**: 与毛发类似，运动步数据使用 `float4` 格式存储在 `ATTR_STD_MOTION_VERTEX_POSITION` 属性中，w 分量存储半径。
- **friend 类**: 声明了 `BVH2`、`BVHBuild`、`BVHSpatialSplit`、`DiagSplit`、`EdgeDice`、`GeometryManager`、`ObjectManager` 为友元类，允许直接访问私有成员。
- **着色器查找优化**: `pack()` 中缓存上一次查找的着色器索引，当连续点使用相同着色器时跳过重复查找。

## 关联文件

- `scene/geometry.h` — 基类
- `scene/object.cpp` — 对象引用点云几何体
- `scene/geometry_mesh.cpp` — 点云数据上传到设备
