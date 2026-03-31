# volume.h / volume.cpp — 体积几何体与体积管理器

## 概述

`volume.h` 和 `volume.cpp` 实现了 Cycles 的体积渲染支持，包括 `Volume` 几何类（继承自 `Mesh`）和 `VolumeManager`（八叉树管理器）。`Volume` 通过将 OpenVDB 体素数据转换为代理三角网格来参与光线追踪。`VolumeManager` 负责构建体积密度八叉树用于空散射（Null Scattering）加速，或在光线步进（Ray Marching）模式下管理步长。

## 类与结构体

### Volume

- **继承**: `Mesh`
- **功能**: 体积几何体，继承 Mesh 以利用其三角网格代理用于光线求交
- **关键成员**:
  - `step_size` (`float`) — 用户指定的步长（0 表示自动检测）
  - `object_space` (`bool`) — 步长是否在物体空间中定义
  - `velocity_scale` (`float`) — 速度场缩放因子（用于运动模糊）
- **关键方法**:
  - `merge_grids()` — 合并分离的速度分量标量网格（velocity_x/y/z）为矢量网格
  - `clear()` — 清除数据（保留体素属性数据）

### VolumeManager

- **功能**: 管理体积渲染的八叉树构建、步长计算和设备同步
- **关键成员**:
  - `object_octrees_` (`map<pair<Object*,Shader*>, shared_ptr<Octree>>`) — 每对象每着色器的八叉树
  - `need_rebuild_` (`bool`) — 是否需要重建八叉树
  - `need_update_step_size` (`bool`) — 是否需要更新步长
  - `last_algorithm` (`VolumeRenderingAlgorithm`) — 上次使用的渲染算法
- **关键方法**:
  - `device_update()` — 设备更新：根据算法选择八叉树构建或步长更新
  - `device_free()` — 释放设备内存
  - `tag_update()` — 多个重载，标记各种变更
  - `is_homogeneous_volume()` — 静态方法，判断体积是否均匀（无空间变化）
  - `initialize_octree()` — 初始化八叉树根节点，同实例共享八叉树
  - `build_octree()` — 在设备上构建八叉树（基于体积密度）
  - `flatten_octree()` — 将八叉树展平为数组上传到内核
  - `update_step_size()` — 计算并更新光线步进步长
  - `visualize_octree()` — 将八叉树导出为 Python 脚本（调试用，需设置 `CYCLES_VOLUME_OCTREE_DUMP` 环境变量）

### VolumeRenderingAlgorithm（枚举）

- `NULL_SCATTERING` — 空散射算法（使用八叉树加速）
- `RAY_MARCHING` — 光线步进算法
- `NONE` — 未初始化

### VolumeMeshBuilder（volume.cpp 内部）

- **功能**: 从 OpenVDB 网格拓扑构建体积代理网格
- **关键方法**:
  - `add_grid()` — 合并 NanoVDB 网格拓扑到 OpenVDB MaskGrid
  - `add_padding()` — 对活跃体素进行膨胀以适应采样填充
  - `create_mesh()` — 生成顶点和三角形索引
  - `generate_vertices_and_quads()` — 光线步进模式下按叶节点生成面片，否则仅生成全局包围盒 6 面

## 核心函数

### 体积网格创建

`GeometryManager::create_volume_mesh()` 流程（定义在 volume.cpp）：
1. 查找体积着色器，确定填充大小（线性插值=1，三次插值=2）
2. 遍历体素属性，创建 NanoVDB 网格句柄
3. 估算速度场所需额外填充
4. 合并所有网格拓扑，执行膨胀
5. 生成顶点（索引空间 -> 物体空间）和三角形
6. 略微偏移顶点以避免面重叠

### 速度场合并

`merge_scalar_grids_for_velocity()` 将三个分量的标量 OpenVDB 网格（velocity_x/y/z）合并为一个 `Vec3fGrid`，并创建对应的体素属性。

### 八叉树流水线

1. `initialize_octree()` — 为每个体积对象+着色器对创建八叉树根节点；非空间变化的着色器共享实例
2. `build_octree()` — 在设备上构建八叉树
3. `flatten_octree()` — 展平为 `KernelOctreeRoot` + `KernelOctreeNode` 数组上传

### SDF 网格生成（WITH_OPENVDB）

`mesh_to_sdf_grid()` 将三角网格转换为 SDF 网格，用于判断空间点是否在网格内部，以减少异构体积的求值区域。要求网格必须是封闭的（`mesh_is_closed` 通过边计数验证）。

## 依赖关系

- **内部头文件**: `graph/node.h`, `scene/mesh.h`, 条件包含 `<openvdb/openvdb.h>`
- **被引用**: `scene/object.cpp`, `scene/geometry.cpp`, `scene/shader.cpp`, `scene/scene.cpp`

## 实现细节 / 关键算法

- **拓扑合并**: 使用 OpenVDB 的 `MaskGrid` 存储纯拓扑信息以节省内存，通过 `topologyUnion()` 合并多个体素网格。
- **面重叠避免**: 使用基于体积名称哈希的随机偏移量，避免多个重叠体积的面片完全重合导致交点歧义。
- **实例共享**: 非空间变化着色器的同几何体实例共享同一个八叉树，节省内存和构建时间。
- **光线步进 vs 空散射**: `device_update()` 根据 `integrator->get_volume_ray_marching()` 选择算法。光线步进不需要八叉树，只需步长信息。

## 关联文件

- `scene/mesh.h` — 基类
- `scene/image_vdb.h` — VDB 图像加载器
- `bvh/octree.h` — 八叉树数据结构
- `scene/integrator.h` — 积分器设置（决定体积渲染算法）
- `scene/object.cpp` — 对象级步长计算
