# multi.h / multi.cpp - 多后端组合 BVH 管理器

## 概述

本文件实现了 Cycles 渲染器的多后端组合层次包围体(BVH)管理器。当渲染任务需要同时使用多种 BVH 后端时(例如 CPU 使用 Embree 而 GPU 使用 OptiX),`BVHMulti` 作为容器持有多个子 BVH 实例,并将几何体替换等操作统一转发到所有子 BVH。典型的多后端布局包括 `BVH_LAYOUT_MULTI_OPTIX_EMBREE`、`BVH_LAYOUT_MULTI_METAL_EMBREE` 等。

## 类与结构体

### BVHMulti
- **继承**: `BVH`（定义于 `bvh/bvh.h`）
- **功能**: 组合多个 BVH 后端实例,统一管理几何体和对象的替换操作。
- **关键成员**:
  - `sub_bvhs` (`vector<unique_ptr<BVH>>`) — 子 BVH 实例列表,每个元素对应一种后端实现（如 BVHEmbree、BVHOptiX 等）
- **关键方法**:
  - `BVHMulti(const BVHParams &params, const vector<Geometry *> &geometry, const vector<Object *> &objects)` — 构造函数,仅初始化基类成员
  - `replace_geometry(const vector<Geometry *> &geometry, const vector<Object *> &objects)` — 重写基类虚方法,将几何体替换操作转发到所有子 BVH

## 核心函数

### `BVHMulti::BVHMulti()`
构造函数仅将参数传递给基类 `BVH` 构造函数,不做额外初始化。`sub_bvhs` 向量初始为空,由外部代码（通常是设备管理层）在构建时填充各后端的子 BVH 实例。

### `BVHMulti::replace_geometry()`
遍历 `sub_bvhs` 中的每个子 BVH,调用其 `replace_geometry()` 方法:
```cpp
for (unique_ptr<BVH> &bvh : sub_bvhs) {
    bvh->replace_geometry(geometry, objects);
}
```
这确保当场景几何体发生变化时,所有后端的 BVH 实例都能同步更新。

## 依赖关系

- **内部头文件**:
  - `bvh/bvh.h` — 基类 `BVH`
  - `bvh/params.h` — `BVHParams`
  - `util/unique_ptr.h` — `unique_ptr`
  - `util/vector.h` — `vector`
- **被引用**:
  - `bvh/bvh.cpp` — `BVH::create()` 工厂方法中为所有 `BVH_LAYOUT_MULTI_*` 布局创建 `BVHMulti` 实例
  - `device/multi/device.cpp` — 多设备管理器填充 `sub_bvhs` 并管理各后端的构建

## 实现细节 / 关键算法

### 组合模式
`BVHMulti` 采用经典的组合(Composite)设计模式。它本身是 `BVH` 的子类,同时包含一个 `BVH` 子类实例的列表。对外表现为单一 BVH 接口,对内将操作分发到多个具体后端。

### 子 BVH 的填充
`BVHMulti` 构造时 `sub_bvhs` 为空。实际的子 BVH 创建由 `device/multi/device.cpp` 中的多设备管理器完成:
1. 为每个设备创建对应布局的子 BVH（如 Embree 用于 CPU、OptiX 用于 NVIDIA GPU）
2. 将子 BVH 的 `unique_ptr` 移入 `sub_bvhs`

### 多后端布局枚举
支持的多后端布局类型(定义于内核类型中):
- `BVH_LAYOUT_MULTI_OPTIX` — OptiX 多设备
- `BVH_LAYOUT_MULTI_METAL` — Metal 多设备
- `BVH_LAYOUT_MULTI_HIPRT` — HIPRT 多设备
- `BVH_LAYOUT_MULTI_EMBREEGPU` — Embree GPU 多设备
- `BVH_LAYOUT_MULTI_OPTIX_EMBREE` — OptiX + Embree 混合
- `BVH_LAYOUT_MULTI_METAL_EMBREE` — Metal + Embree 混合
- `BVH_LAYOUT_MULTI_HIPRT_EMBREE` — HIPRT + Embree 混合
- `BVH_LAYOUT_MULTI_EMBREEGPU_EMBREE` — Embree GPU + Embree CPU 混合

## 关联文件

- `bvh/bvh.h` / `bvh/bvh.cpp` — BVH 基类和工厂方法
- `bvh/embree.h` / `bvh/embree.cpp` — 可能的子 BVH 后端之一
- `bvh/optix.h` / `bvh/optix.cpp` — 可能的子 BVH 后端之一
- `bvh/metal.h` / `bvh/metal.mm` — 可能的子 BVH 后端之一
- `bvh/hiprt.h` / `bvh/hiprt.cpp` — 可能的子 BVH 后端之一
- `device/multi/device.cpp` — 多设备管理器,负责填充子 BVH
