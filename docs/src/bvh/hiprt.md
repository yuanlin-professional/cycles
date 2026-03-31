# hiprt.h / hiprt.cpp - AMD HIPRT 光线追踪后端集成

## 概述

本文件实现了 Cycles 渲染器与 AMD HIPRT (HIP Ray Tracing) 库的集成。`BVHHIPRT` 类继承自 `BVH` 基类,提供在 AMD GPU 上使用硬件加速射线追踪构建 BVH 所需的数据结构和设备内存管理。该实现以数据持有为主,实际的 BVH 构建和射线遍历逻辑委托给 HIPDevice 完成。

## 类与结构体

### BVHHIPRT
- **继承**: `BVH`（定义于 `bvh/bvh.h`）
- **功能**: 持有 HIPRT 几何体句柄和相关的设备端内存缓冲区,作为 AMD HIP 设备与 Cycles BVH 系统之间的桥梁。
- **关键成员**:
  - `hiprt_geom` (`hiprtGeometry`) — HIPRT 几何体句柄
  - `triangle_mesh` (`hiprtTriangleMeshPrimitive`) — 三角形网格图元描述
  - `custom_prim_aabb` (`hiprtAABBListPrimitive`) — 自定义图元的 AABB 列表（用于曲线、点云等非三角形图元）
  - `geom_input` (`hiprtGeometryBuildInput`) — HIPRT 几何体构建输入参数
  - `custom_prim_info` (`vector<int2>`) — 自定义图元信息,x 为图元 ID,y 为图元类型
  - `prims_time` (`vector<float2>`) — 图元时间范围（用于运动模糊）
  - `custom_primitive_bound` (`device_vector<BoundBox>`) — 自定义图元包围盒(设备端只读内存)
  - `triangle_index` (`device_vector<int>`) — 三角形索引缓冲区(设备端只读内存)
  - `vertex_data` (`device_vector<float>`) — 顶点数据缓冲区(设备端只读内存)
  - `device` (`Device *`) — 关联的 HIP 设备指针
- **关键方法**:
  - `BVHHIPRT(const BVHParams &params, const vector<Geometry *> &geometry, const vector<Object *> &objects, Device *in_device)` — 构造函数,初始化设备内存缓冲区
  - `~BVHHIPRT()` — 析构函数,释放设备内存并通知设备释放 BVH 资源
  - `is_tlas()` — 判断是否为顶层加速结构(Top-Level Acceleration Structure)

## 核心函数

### 构造函数 `BVHHIPRT::BVHHIPRT()`
1. 调用基类 `BVH` 构造函数
2. 初始化 `hiprt_geom` 为 `nullptr`
3. 以只读模式分配三个设备内存缓冲区:
   - `custom_primitive_bound` — "Custom Primitive Bound"
   - `triangle_index` — "HIPRT Triangle Index"
   - `vertex_data` — "vertex_data"
4. 将 `triangle_mesh` 和 `custom_prim_aabb` 内部指针置空

### 析构函数 `BVHHIPRT::~BVHHIPRT()`
1. 释放三个设备内存缓冲区(`free()`)
2. 调用 `device->release_bvh(this)` 通知设备端释放相关的 HIPRT BVH 资源

## 依赖关系

- **内部头文件**:
  - `bvh/bvh.h` — 基类 `BVH`
  - `bvh/params.h` — `BVHParams`
  - `device/memory.h` — `device_vector` 设备内存向量
  - `scene/mesh.h`、`scene/object.h` — 场景几何体
  - `device/hiprt/device_impl.h` — HIP 设备实现
- **外部依赖**:
  - `hiprtew.h`（动态加载）或 `hiprt/hiprt_types.h`（静态链接）— HIPRT 类型定义
- **被引用**:
  - `bvh/bvh.cpp` — 工厂方法中创建 `BVHHIPRT` 实例（条件编译 `WITH_HIPRT`）
  - `device/hiprt/device_impl.h` — `HIPDevice` 作为友元类访问 BVHHIPRT 的内部成员

## 实现细节 / 关键算法

### TLAS vs BLAS 区分
通过 `params.top_level` 判断当前 BVH 是顶层加速结构(TLAS)还是底层加速结构(BLAS)。TLAS 包含实例引用,BLAS 包含实际的几何图元。

### 设备内存管理
使用 `device_vector` 模板管理设备端内存,所有缓冲区以 `MEM_READ_ONLY` 模式分配,表示数据仅从主机写入、设备端只读。析构时显式调用 `free()` 释放设备内存。

### 自定义图元支持
非三角形图元(如曲线、点云)通过 HIPRT 的 `hiprtAABBListPrimitive` 接口以自定义 AABB 列表形式提供,每个自定义图元附带 `int2` 类型的信息(图元 ID 和类型),供相交测试着色器使用。

### 构建委托
与 `BVHEmbree` 不同,`BVHHIPRT` 本身不包含 `build()` 方法。实际的 HIPRT BVH 构建由 `HIPDevice`（通过友元关系直接访问 `BVHHIPRT` 的成员）在设备端执行。这种设计将 BVH 数据持有和构建逻辑分离。

## 关联文件

- `bvh/bvh.h` / `bvh/bvh.cpp` — BVH 基类和工厂方法
- `bvh/params.h` — BVH 参数定义
- `device/hiprt/device_impl.h` / `device/hiprt/device_impl.cpp` — HIP 设备实现,执行实际 HIPRT BVH 构建
- `device/memory.h` — 设备内存管理
