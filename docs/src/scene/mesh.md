# mesh.h / mesh.cpp — 三角网格几何体

## 概述

`mesh.h` 和 `mesh.cpp` 实现了 Cycles 的核心三角网格类 `Mesh`，它继承自 `Geometry` 基类。该类管理三角形顶点、面数据、细分曲面（Subdivision Surface）数据，以及网格级别的 BVH 偏移量。`Mesh` 是 `Volume` 类的基类，同时支持 Catmull-Clark 和线性细分。

额外实现分布在 `mesh_displace.cpp`（位移着色器求值）和 `mesh_subdivision.cpp`（曲面细分/切分/骰子化）中。

## 类与结构体

### Mesh

- **继承**: `Geometry`
- **功能**: 三角网格的核心数据结构，支持直接三角形和细分曲面两种输入模式
- **关键成员**:
  - `triangles` (`array<int>`) — 三角形顶点索引（每 3 个元素一个三角形）
  - `verts` (`array<float3>`) — 顶点坐标
  - `shader` (`array<int>`) — 每三角形着色器索引
  - `smooth` (`array<bool>`) — 每三角形平滑着色标志
  - `subdivision_type` (`SubdivisionType`) — 细分类型：`NONE` / `LINEAR` / `CATMULL_CLARK`
  - `subd_face_corners` (`array<int>`) — 细分面角点顶点索引
  - `subd_start_corner` / `subd_num_corners` / `subd_shader` / `subd_smooth` / `subd_ptex_offset` — 细分面属性数组
  - `subd_creases_edge` / `subd_creases_weight` — 边折痕数据
  - `subd_vert_creases` / `subd_vert_creases_weight` — 顶点折痕数据
  - `subd_dicing_rate` (`float`) — 细分切分率
  - `subd_max_level` (`int`) — 最大细分层级
  - `subd_attributes` (`AttributeSet`) — 细分专用属性集
  - `vert_offset` / `face_offset` / `corner_offset` (`size_t`) — 全局数组中的偏移
- **关键方法**:
  - `resize_mesh()` / `reserve_mesh()` — 调整/预分配网格数据
  - `resize_subd_faces()` / `reserve_subd_faces()` — 调整/预分配细分面数据
  - `add_vertex()` / `add_vertex_slow()` — 添加顶点（预分配/慢速模式）
  - `add_triangle()` — 添加三角形
  - `add_subd_face()` — 添加细分面（支持 N 边形）
  - `add_edge_crease()` / `add_vertex_crease()` — 添加边/顶点折痕
  - `compute_bounds()` — 计算包围盒，含运动模糊步
  - `apply_transform()` — 对顶点和法线应用变换矩阵
  - `add_vertex_normals()` — 计算静态和运动法线
  - `add_undisplaced()` — 保存位移前位置和法线
  - `update_generated()` — 创建 generated 属性
  - `update_tangents()` — 通过 MikkTSpace 算法计算切线
  - `pack_shaders()` / `pack_normals()` / `pack_verts()` — 将数据打包为设备格式
  - `tessellate()` — 执行曲面细分
  - `need_tesselation()` — 判断是否需要重新细分
  - `has_motion_blur()` — 检查运动模糊状态
  - `primitive_type()` — 返回 `PRIMITIVE_TRIANGLE` 或 `PRIMITIVE_MOTION_TRIANGLE`

### Mesh::Triangle

- **功能**: 三角形辅助结构
- **关键成员**: `v[3]` — 三个顶点索引
- **关键方法**:
  - `bounds_grow()` — 扩展包围盒
  - `motion_verts()` — 在运动模糊步之间插值顶点
  - `verts_for_step()` — 获取特定运动步的顶点
  - `compute_normal()` — 计算面法线
  - `valid()` — 检查顶点坐标是否有效（非 NaN/Inf）

### Mesh::SubdFace

- **功能**: 细分面辅助结构
- **关键成员**: `start_corner`, `num_corners`, `shader`, `smooth`, `ptex_offset`
- **关键方法**:
  - `is_quad()` — 是否为四边形
  - `normal()` — 计算面法线
  - `num_ptex_faces()` — PTex 面数（四边形=1，N边形=N）

### Mesh::SubdEdgeCrease

- **功能**: 边折痕结构，包含两端顶点索引 `v[2]` 和折痕权重 `crease`

### MikkMeshWrapper（mesh.cpp 内部）

- **功能**: MikkTSpace 切线计算的网格适配器，将 Cycles 网格接口包装为 MikkTSpace 所需的回调接口

## 核心函数

### 切线计算（mesh.cpp）

`mikk_compute_tangents()` 使用 MikkTSpace 库为指定 UV 贴图计算切线和副切线符号，支持标准 UV 和自定义 UV 属性。

### 位移（mesh_displace.cpp）

`GeometryManager::displace()` 流程：
1. 调用 `add_undisplaced()` 保存位移前坐标
2. 通过 `fill_shader_input()` 准备着色器求值输入
3. 在 GPU 上执行位移着色器 (`SHADER_EVAL_DISPLACE`)
4. `read_shader_output()` 读取位移偏移并应用到顶点
5. 对使用真实位移（非凹凸）的三角形重新计算法线
6. 更新切线

### 曲面细分（mesh_subdivision.cpp）

`Mesh::tessellate()` 流程：
1. 统计 Patch 数量（四边形=1 patch，N边形=N patches）
2. Catmull-Clark 模式：使用 OpenSubdiv 构建 `OsdPatch`
3. 线性模式：构建 `LinearQuadPatch`（N边形分割为扇形四边形）
4. `DiagSplit` 执行自适应分割（基于像素空间或物体空间的 dicing rate）
5. `EdgeDice` 将分割后的 Patch 骰子化为三角形
6. `SubdAttributeInterpolation` 处理属性插值

## 依赖关系

- **内部头文件**: `graph/node.h`, `scene/attribute.h`, `scene/geometry.h`, `scene/shader.h`, `subd/dice.h`, `util/array.h`, `util/boundbox.h`
- **被引用**: `scene/volume.h`, `scene/object.cpp`, `scene/geometry.cpp`, `scene/geometry_mesh.cpp`, `scene/light.cpp`, `scene/bake.cpp`, `scene/camera.cpp` 等大量文件

## 实现细节 / 关键算法

- **法线计算**: 面法线通过叉积得到后累加到顶点法线，最终归一化。负缩放时取反。运动模糊的每步法线独立计算。
- **细分面支持**: N 边形通过 `subd_face_corners` + `subd_start_corner` + `subd_num_corners` 组合存储，支持任意边数多边形。
- **PTex 偏移**: 每个四边形对应 1 个 PTex 面，N 边形对应 N 个 PTex 面，偏移量逐面累计。
- **运动模糊顶点**: 中心步存储在 `verts` 中，其余步存储在 `ATTR_STD_MOTION_VERTEX_POSITION` 属性中。查找时需要跳过中心步的空位。
- **包围盒计算**: 先尝试所有顶点（含运动步），如果出现无效坐标则退回到 `grow_safe` 模式跳过 NaN/Inf。

## 关联文件

- `mesh_displace.cpp` — 位移着色器在设备上的求值
- `mesh_subdivision.cpp` — 曲面细分实现
- `scene/geometry.h` — 基类
- `scene/volume.h` — Volume 子类继承 Mesh
- `scene/attribute.h` — 属性系统
- `subd/split.h`, `subd/dice.h`, `subd/osd.h`, `subd/patch.h` — 细分相关模块
