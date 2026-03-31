# attribute.h / attribute.cpp — 属性系统

## 概述

`attribute.h` 和 `attribute.cpp` 实现了 Cycles 的几何属性系统，支持在网格、毛发、点云等几何体上存储任意数据层（如 UV、法线、颜色、速度等）。属性系统由四个核心类组成：`Attribute`（单个属性）、`AttributeSet`（属性集合）、`AttributeRequest`（着色器属性请求）和 `AttributeRequestSet`（请求集合）。

## 类与结构体

### Attribute

- **功能**: 几何体上的一个数据层，支持多种数据类型和元素级别
- **关键成员**:
  - `name` (`ustring`) — 属性名称
  - `std` (`AttributeStandard`) — 标准属性类型（如 `ATTR_STD_UV`、`ATTR_STD_VERTEX_NORMAL`）
  - `type` (`TypeDesc`) — 数据类型（Float、Float2、Float3、Float4、Color、Matrix 等）
  - `buffer` (`vector<char>`) — 原始数据缓冲区
  - `element` (`AttributeElement`) — 元素级别（`ATTR_ELEMENT_VERTEX`、`ATTR_ELEMENT_FACE`、`ATTR_ELEMENT_CORNER`、`ATTR_ELEMENT_CURVE`、`ATTR_ELEMENT_VOXEL` 等）
  - `flags` (`uint`) — 属性标志
  - `modified` (`bool`) — 是否已修改
- **关键方法**:
  - `set()` — 设置属性名称、类型和元素级别
  - `resize()` — 根据几何体大小调整缓冲区
  - `data()` / `data_float()` / `data_float2()` / `data_float3()` / `data_float4()` / `data_uchar4()` / `data_transform()` / `data_voxel()` — 类型化数据访问器
  - `data_sizeof()` — 单个元素的字节大小
  - `element_size()` — 根据几何体和元素级别计算元素数量
  - `buffer_size()` — 总缓冲区大小
  - `zero_data()` — 零初始化
  - `add()` — 追加数据（多种类型重载）
  - `set_data_from()` — 从另一个属性移动数据（仅在数据不同时标记修改）
  - `same_storage()` — 静态方法，检查两种类型是否使用相同的设备存储
  - `standard_name()` / `name_standard()` — 标准属性名称与枚举之间的转换
  - `kernel_type()` — 返回属性在内核中的数据类型（`AttrKernelDataType`）
  - `get_uv_tiles()` — 从 UV 属性中提取 UDIM 瓦片索引

### AttrKernelDataType（枚举）

设备数组中存储属性数据的类型，用于变更检测和内存管理：
- `FLOAT` = 0, `FLOAT2` = 1, `FLOAT3` = 2, `FLOAT4` = 3, `UCHAR4` = 4, `NUM` = 5

### AttributeSet

- **功能**: 几何体上的属性集合，管理属性的添加、查找、删除和设备同步
- **关键成员**:
  - `geometry` (`Geometry*`) — 所属几何体
  - `prim` (`AttributePrimitive`) — 图元类型（`ATTR_PRIM_GEOMETRY` 或 `ATTR_PRIM_SUBD`）
  - `attributes` (`list<Attribute>`) — 属性链表
  - `modified_flag` (`uint32_t`) — 修改标志位
- **关键方法**:
  - `add(ustring, TypeDesc, AttributeElement)` — 按名称添加自定义属性
  - `add(AttributeStandard)` — 按标准类型添加属性
  - `find(ustring)` / `find(AttributeStandard)` — 查找属性
  - `remove()` — 按名称、标准类型或迭代器删除属性
  - `copy()` — 复制一个属性
  - `resize()` — 调整所有属性的缓冲区大小
  - `clear()` — 清除所有属性（可选保留体素数据）
  - `update()` — 用新属性集更新当前集合（差异更新）
  - `modified()` — 检查指定内核类型的属性是否有结构性变更（添加/删除）
  - `clear_modified()` — 清除修改标志
  - `tag_modified()` — 标记指定属性类型已修改

### AttributeRequest

- **功能**: 着色器对属性的请求，通过名称或标准类型标识
- **关键成员**:
  - `name` (`ustring`) — 属性名称
  - `std` (`AttributeStandard`) — 标准属性类型
  - `type` (`TypeDesc`) — 解析后的属性类型（由 `GeometryManager` 填充）
  - `desc` (`AttributeDescriptor`) — 内核描述符（由 `GeometryManager` 填充）

### AttributeRequestSet

- **功能**: 着色器请求的属性集合
- **关键成员**: `requests` (`vector<AttributeRequest>`) — 请求列表
- **关键方法**:
  - `add(ustring)` / `add(AttributeStandard)` / `add(AttributeRequestSet&)` — 添加请求
  - `add_standard(ustring)` — 通过名称查找标准属性类型并添加
  - `find()` — 查找请求
  - `modified()` — 与另一个请求集比较是否有变化

## 核心函数

### 属性缓冲区管理

`Attribute::resize()` 根据几何体类型和元素级别计算所需元素数量：
- `ATTR_ELEMENT_VERTEX` — 顶点数（Mesh）/ 控制点数（Hair）/ 点数（PointCloud）
- `ATTR_ELEMENT_FACE` — 三角形数 / 曲线数
- `ATTR_ELEMENT_CORNER` — 三角形数 x 3
- `ATTR_ELEMENT_CURVE_KEY` — 控制点数
- `ATTR_ELEMENT_VOXEL` — 存储 `ImageHandle` 而非数据数组

### 增量更新

`AttributeSet::update()` 执行差异更新：
1. 遍历新属性集中的属性
2. 如果旧集合中存在同名同类型属性，调用 `set_data_from()` 仅在数据变化时标记修改
3. 删除新集合中不存在的旧属性
4. 添加新集合中新增的属性

## 依赖关系

- **内部头文件**: `scene/image.h`, `kernel/types.h`, `util/list.h`, `util/param.h`, `util/set.h`, `util/types.h`, `util/vector.h`
- **被引用**: `scene/mesh.h`, `scene/geometry.h`, `scene/geometry_attributes.cpp`, `scene/geometry_mesh.cpp`, `scene/volume.cpp`, `scene/shader.h` 等

## 实现细节 / 关键算法

- **体素属性**: `ATTR_ELEMENT_VOXEL` 类型的属性缓冲区存储 `ImageHandle` 而非实际数据，图像数据由图像管理器独立管理。析构时需要显式调用 `ImageHandle` 析构函数。
- **数据类型映射**: `kernel_type()` 将属性类型映射到内核存储类型：Point/Vector/Normal/Transform -> FLOAT3, Color/RGBA -> FLOAT4, Float2 -> FLOAT2, uchar4 -> UCHAR4, Float -> FLOAT。
- **同存储判断**: `same_storage()` 检查两种类型是否可以共用同一设备数组（如 Point 和 Vector 都存储为 float3）。
- **标准属性名称**: `standard_name()` 提供从枚举到字符串的映射（如 `ATTR_STD_UV` -> "std_uv"），用于在不同系统间匹配属性。

## 关联文件

- `scene/geometry.h` — `Geometry` 类持有 `AttributeSet`
- `scene/mesh.h` — `Mesh` 额外持有 `subd_attributes`（细分属性集）
- `scene/geometry_attributes.cpp` — 属性上传到设备的具体实现
- `kernel/types.h` — 内核中的属性描述符定义
