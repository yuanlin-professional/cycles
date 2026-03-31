# bvh.h / bvh.cpp - BVH 基类与工厂方法,以及打包数据结构

## 概述

本文件定义了 Cycles 渲染器中所有层次包围体(BVH)实现的基类 `BVH` 以及设备端使用的打包数据结构 `PackedBVH`。`BVH` 类通过工厂方法 `BVH::create()` 根据请求的布局类型(BVH2、Embree、OptiX、Metal、HIPRT、Multi 等)创建对应的子类实例。`bvh.cpp` 还实现了 `BVHParams::best_bvh_layout()` 用于在设备支持的布局中选择最优匹配,以及 `bvh_layout_name()` 用于获取布局的人类可读名称。

## 类与结构体

### PackedBVH
- **功能**: 存储最终供渲染设备遍历使用的打包 BVH 数据。
- **关键成员**:
  - `nodes` (`array<int4>`) — BVH 内部节点存储,每个节点 4 个 `int4`,包含两个子包围盒及子节点/三角形/对象索引
  - `leaf_nodes` (`array<int4>`) — BVH 叶节点存储
  - `object_node` (`array<int>`) — 对象索引到 BVH 节点索引的映射（用于实例化）
  - `prim_type` (`array<int>`) — 图元类型（三角形或线段曲线）
  - `prim_visibility` (`array<uint>`) — 图元可见性标志
  - `prim_index` (`array<int>`) — BVH 图元索引到真实图元索引的映射（空间分割可能导致重复,-1 表示实例）
  - `prim_object` (`array<int>`) — BVH 图元索引到所属对象 ID 的映射
  - `prim_time` (`array<float2>`) — BVH 图元的时间范围
  - `root_index` (`int`) — 根节点索引

### BVH
- **继承**: 无(是所有具体 BVH 实现的基类)
- **功能**: 定义 BVH 的通用接口和工厂方法。
- **关键成员**:
  - `params` (`BVHParams`) — BVH 构建参数
  - `geometry` (`vector<Geometry *>`) — 关联的几何体列表
  - `objects` (`vector<Object *>`) — 关联的对象列表
- **关键方法**:
  - `static create(const BVHParams &params, const vector<Geometry *> &geometry, const vector<Object *> &objects, Device *device)` — 工厂方法,根据 `params.bvh_layout` 创建对应子类
  - `replace_geometry(...)` — 替换关联的几何体和对象(虚方法,子类可重写)

## 核心函数

### `BVH::create()`
根据 `params.bvh_layout` 分发创建:
- `BVH_LAYOUT_BVH2` -> `BVH2`
- `BVH_LAYOUT_EMBREE` / `BVH_LAYOUT_EMBREEGPU` -> `BVHEmbree`（需 `WITH_EMBREE` 编译宏）
- `BVH_LAYOUT_OPTIX` -> `BVHOptiX`（需 `WITH_OPTIX` 编译宏）
- `BVH_LAYOUT_METAL` -> `bvh_metal_create()`（需 `WITH_METAL` 编译宏）
- `BVH_LAYOUT_HIPRT` -> `BVHHIPRT`（需 `WITH_HIPRT` 编译宏）
- `BVH_LAYOUT_MULTI_*` -> `BVHMulti`

### `BVHParams::best_bvh_layout()`
在设备支持的布局掩码中选择最优 BVH 布局:
1. 若请求的布局被支持,直接返回
2. 否则选择支持的布局中最宽的(通过位操作 `__bsr` 找到最高有效位)
3. 优先选择不超过请求布局的最宽布局

### `bvh_layout_name()`
将 `BVHLayout` 枚举值转换为人类可读的字符串名称("BVH2"、"EMBREE"、"OPTIX"、"METAL"、"HIPRT"、"MULTI" 等)。

## 依赖关系

- **内部头文件**:
  - `bvh/params.h` — `BVHParams`、`BVHLayout`、`BVHLayoutMask`
  - `bvh/bvh2.h` — `BVH2` 子类
  - `bvh/multi.h` — `BVHMulti` 子类
  - `bvh/embree.h` — `BVHEmbree` 子类（条件编译）
  - `bvh/hiprt.h` — `BVHHIPRT` 子类（条件编译）
  - `bvh/metal.h` — `bvh_metal_create` 工厂函数（条件编译）
  - `bvh/optix.h` — `BVHOptiX` 子类（条件编译）
  - `util/array.h`、`util/types.h`、`util/vector.h`、`util/unique_ptr.h` — 基础容器和类型
- **被引用**:
  - `bvh/bvh2.h` — BVH2 继承 BVH
  - `bvh/embree.h` — BVHEmbree 继承 BVH
  - `bvh/multi.h` — BVHMulti 继承 BVH
  - `bvh/node.cpp` — 引用 BVH 头文件
  - `scene/geometry.cpp`、`scene/geometry_bvh.cpp`、`scene/scene.cpp` — 场景管理中创建和管理 BVH
  - `scene/mesh.cpp`、`scene/hair.cpp`、`scene/pointcloud.cpp` — 各几何体类型的 BVH 管理
  - `device/device.cpp` — 设备级 BVH 管理

## 实现细节 / 关键算法

### BVH 布局选择
`best_bvh_layout` 使用位掩码操作实现高效的布局匹配。所有 BVH 布局以 2 的幂次表示,构成自然的位掩码体系。"最宽" 布局意味着更高级的硬件加速(如 OptiX > Embree > BVH2),通过 `__bsr`（位扫描反向）找到最高有效位来确定。

### 工厂模式
`BVH::create()` 采用经典工厂模式,将具体子类的创建逻辑集中管理。各后端(Embree、OptiX、Metal、HIPRT)通过条件编译宏控制可用性。

### `PackedBVH` 内存布局
BVH 节点以紧凑的 `int4` 数组存储,每个内部节点占 4 个 `int4`(对齐节点)或 7 个 `int4`(非轴对齐节点)。叶节点和内部节点分开存储,叶节点索引通过按位取反(~idx)与内部节点索引区分。

## 关联文件

- `bvh/bvh2.h` / `bvh/bvh2.cpp` — 二叉 BVH 具体实现
- `bvh/embree.h` / `bvh/embree.cpp` — Embree 后端
- `bvh/multi.h` / `bvh/multi.cpp` — 多后端组合 BVH
- `bvh/hiprt.h` / `bvh/hiprt.cpp` — HIPRT 后端
- `bvh/metal.h` / `bvh/metal.mm` — Metal 后端
- `bvh/optix.h` / `bvh/optix.cpp` — OptiX 后端
- `bvh/params.h` — BVH 参数和布局定义
