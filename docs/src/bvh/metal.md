# metal.h / metal.mm - Apple Metal 光线追踪后端工厂接口

## 概述

本文件提供了 Cycles 渲染器中 Apple Metal 光线追踪后端的工厂函数接口。`metal.h` 声明了 `bvh_metal_create()` 工厂函数,`metal.mm`（Objective-C++ 源文件）实现该函数,创建并返回 `BVHMetal` 实例。实际的 `BVHMetal` 类定义和完整实现位于 `device/metal/bvh.h`,本文件仅作为 BVH 子系统与 Metal 设备子系统之间的桥接层。

## 类与结构体

本文件未定义新的类或结构体。

## 核心函数

### `bvh_metal_create()`
```cpp
unique_ptr<BVH> bvh_metal_create(const BVHParams &params,
                                 const vector<Geometry *> &geometry,
                                 const vector<Object *> &objects,
                                 Device *device);
```
- **功能**: 工厂函数,创建 `BVHMetal` 实例
- **参数**:
  - `params` — BVH 构建参数
  - `geometry` — 几何体列表
  - `objects` — 对象列表
  - `device` — Metal 设备指针
- **返回**: `unique_ptr<BVH>` 指向新创建的 `BVHMetal` 实例
- **实现**: 直接调用 `make_unique<BVHMetal>(params, geometry, objects, device)`

## 依赖关系

- **内部头文件**:
  - `bvh/bvh.h` — 基类 `BVH`
  - `util/unique_ptr.h` — `unique_ptr` 和 `make_unique`
  - `device/metal/bvh.h` — `BVHMetal` 类定义（仅 .mm 文件引用）
- **被引用**:
  - `bvh/bvh.cpp` — `BVH::create()` 工厂方法中在 `BVH_LAYOUT_METAL` 分支调用（条件编译 `WITH_METAL`）

## 实现细节 / 关键算法

### Objective-C++ 桥接
Metal API 是基于 Objective-C 的框架,因此实现文件使用 `.mm` 扩展名(Objective-C++ 源文件),以便同时使用 C++ 和 Objective-C 语法。头文件 `metal.h` 保持纯 C++ 接口,使得 BVH 子系统的其他 C++ 代码可以在不涉及 Objective-C 的情况下引用该工厂函数。

### 职责分离
`BVHMetal` 的完整实现位于 `device/metal/bvh.h`,而非 `bvh/` 目录下。这种设计将 Metal 特定的设备逻辑(如 MPSRayIntersector、MTLAccelerationStructure 等)保留在设备子系统中,BVH 子系统仅提供工厂入口点。

### 条件编译
整个文件受 `#ifdef WITH_METAL` 保护。在非 macOS 平台上,该文件完全被跳过。CMakeLists.txt 中通过 `WITH_CYCLES_DEVICE_METAL` 选项控制是否编译 `metal.mm`。

## 关联文件

- `bvh/bvh.h` / `bvh/bvh.cpp` — BVH 基类和工厂方法
- `device/metal/bvh.h` — `BVHMetal` 类的完整定义和实现
- `bvh/CMakeLists.txt` — 条件编译 `metal.mm` 的构建配置
