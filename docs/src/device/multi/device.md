# device.h / device.cpp - 多设备聚合管理器实现

## 概述

本文件实现了 Cycles 渲染器的多设备管理器（MultiDevice），负责将多个物理渲染设备（GPU 和/或 CPU）聚合为一个统一的逻辑设备。MultiDevice 通过对等岛（Peer Island）机制管理设备间的内存共享和分配策略，并将内核加载、BVH 加速结构构建、内存操作等请求分发到所有子设备，实现多设备协同路径追踪渲染。

## 类与结构体

### MultiDevice
- **继承**: `Device`（定义于 `device/device.h`）
- **功能**: 聚合管理多个子设备，对外暴露统一的 `Device` 接口。负责子设备的创建与排序、对等内存访问检测、BVH 构建分发、内存分配与映射管理。

- **关键成员**:
  - `devices` (`list<SubDevice>`) — 子设备链表。使用链表确保插入新设备后已有指针不失效。GPU 设备排在前端，CPU 设备排在后端。
  - `unique_key` (`device_ptr`) — 全局唯一的内存键生成器，从 1 开始递增，用于在多设备间统一标识同一块逻辑内存。
  - `peer_islands` (`vector<vector<SubDevice *>>`) — 对等岛列表。同一岛内的设备支持 P2P（点对点）内存直接访问，可共享内存分配而无需复制。

- **关键方法**:
  - `verify_hardware_raytracing()` — 检查所有 GPU 子设备是否均支持硬件光线追踪；若存在混合情况则统一禁用，因为后端和场景更新代码不支持 BVH2 与硬件光追混合使用。
  - `get_bvh_layout_mask()` — 计算所有子设备 BVH 布局的交集/并集，返回合适的多设备 BVH 布局类型（如 `BVH_LAYOUT_MULTI_OPTIX`、`BVH_LAYOUT_MULTI_METAL` 等）。
  - `load_kernels()` — 遍历所有子设备加载内核，任一失败则整体失败。
  - `load_osl_kernels()` — 遍历所有子设备加载 OSL 内核。
  - `build_bvh()` — 为每个子设备构建独立的 BVH 加速结构，或在共享布局（BVH2/Embree）时复用单一结构。
  - `mem_alloc()` — 按对等岛分配内存，在每个岛内选择内存使用最少的设备进行分配。
  - `mem_copy_to()` — 将数据拷贝到各对等岛的所有者设备，对纹理和全局内存还会在岛内其他设备上更新指针。
  - `mem_copy_from()` — 将数据从设备拷贝回主机，按子设备数量均分高度区域进行并行读回。
  - `mem_free()` — 释放所有对等岛中的内存，并清理指针映射表。
  - `find_matching_mem_device()` — 根据内存键查找所有者子设备（优先当前设备，再查对等岛）。
  - `find_suitable_mem_device()` — 为新的内存分配选择合适的子设备（已有键则返回所有者，新键则选内存使用最少的设备）。
  - `foreach_device()` — 遍历所有子设备并对每个子设备递归执行回调函数。
  - `get_cpu_osl_memory()` — 返回 CPU 子设备的 OSL 全局内存（CPU 设备始终在链表尾部）。
  - `device_number()` — 返回指定子设备在设备列表中的索引编号。

### SubDevice
- **功能**: 封装单个子设备的实例及其关联状态，作为 `MultiDevice` 的内部结构体。
- **关键成员**:
  - `stats` (`Stats`) — 该子设备独立的内存统计信息。
  - `device` (`unique_ptr<Device>`) — 实际子设备实例的所有权智能指针。
  - `ptr_map` (`map<device_ptr, device_ptr>`) — 将 MultiDevice 的全局唯一键映射到该子设备上的实际设备指针。
  - `peer_island_index` (`int`) — 该子设备所属的对等岛索引，初始值 `-1` 表示未分配。

## 核心函数

### `device_multi_create()`
```cpp
unique_ptr<Device> device_multi_create(const DeviceInfo &info,
                                       Stats &stats,
                                       Profiler &profiler,
                                       bool headless);
```
- **功能**: 工厂函数，创建并返回一个 `MultiDevice` 实例。
- **调用时机**: 在 `Device::create()` 中，当 `DeviceInfo::multi_devices` 非空时被调用。即使设备类型标记为 `DEVICE_CPU`，只要包含多个子设备信息就会创建多设备管理器。
- **参数说明**:
  - `info` — 设备信息，其 `multi_devices` 字段包含所有子设备的 `DeviceInfo`。
  - `stats` — 顶层统计信息对象引用。
  - `profiler` — 性能分析器引用。
  - `headless` — 是否为无头模式。

## 依赖关系

- **内部头文件**:
  - `device/multi/device.h` — 本模块头文件，声明 `device_multi_create()` 工厂函数。
  - `device/device.h` — 基类 `Device`、`DeviceInfo`、`DeviceType` 枚举等核心定义。
  - `device/queue.h` — 设备队列定义。
  - `bvh/multi.h` — `BVHMulti` 类，多设备 BVH 容器，持有每个子设备对应的子 BVH。
  - `scene/geometry.h` — `Geometry` 类，BVH 构建时需要访问几何体的 BVH 所有权。
  - `util/list.h` — 链表容器。
  - `util/map.h` — 有序映射容器。
  - `<cstdlib>` — 标准库。
  - `<functional>` — `std::function` 支持 `foreach_device` 回调。
- **被引用**:
  - `src/device/device.cpp` — 在 `Device::create()` 工厂方法中，当检测到 `info.multi_devices` 非空时调用 `device_multi_create()`。

## 实现细节 / 关键算法

### 子设备排序策略
构造函数中，GPU 设备通过 `emplace_front()` 插入链表头部，CPU 设备通过 `emplace_back()` 插入尾部。这一设计确保 GPU 设备优先处理内存操作（因为 GPU 可能改变主机内存指针，而 CPU 直接使用主机指针作为设备指针）。同时保证 CPU 设备始终在链表末尾，使 `get_cpu_osl_memory()` 可以直接访问 `devices.back()`。

### 对等岛（Peer Island）机制
对等岛将支持 P2P 内存直接访问的同类型设备分组：
1. 遍历所有子设备，为每个尚未分配岛的设备创建新岛。
2. 若 `info.has_peer_memory` 为 `true`，检查同类型设备间的 `check_peer_access()` 结果，将可互相访问的设备归入同一岛。
3. 内存分配和管理均以岛为单位进行——同一岛内的设备共享同一份内存，不同岛各自独立分配。

### 内存管理与指针映射
MultiDevice 维护全局唯一键（`unique_key`），对外暴露统一的 `device_ptr`：
- **分配**: 对每个对等岛，在内存使用最少的设备上分配，并在 `ptr_map` 中记录全局键到实际设备指针的映射。
- **拷贝**: 对全局内存和纹理内存，除所有者设备外还需在岛内其他设备上更新纹理对象和内核全局指针。
- **释放**: 遍历所有岛，查找并释放内存，清理映射表条目。纹理类型还需在岛内非所有者设备上额外释放。
- **回读**: `mem_copy_from()` 将高度 `h` 均分给各子设备，实现分块并行回读。

### BVH 构建分发
`build_bvh()` 的逻辑分为两种情况：
1. **共享布局**（BVH2 或 Embree）: 仅在最后一个设备（CPU）上构建一次，所有设备共享。
2. **多设备布局**（Multi-OptiX/Metal/HIP/oneAPI 等）: 使用 `BVHMulti` 容器，为每个子设备构建独立的加速结构。构建过程中临时转移几何体 BVH 所有权至各子 BVH，构建完成后恢复。对于 Embree 后端的非实例化几何体，跳过底层加速结构构建（因为它们直接放入顶层结构）。

### 硬件光线追踪一致性校验
`verify_hardware_raytracing()` 确保所有 GPU 子设备的硬件光追设置一致。若存在部分启用、部分禁用的混合情况，将统一禁用硬件光追，因为混合模式在后端和场景更新代码中不受支持。

## 关联文件

- `src/device/device.h` — `Device` 基类定义，包含 `DeviceInfo`、`DeviceType` 枚举及所有虚函数接口。
- `src/device/device.cpp` — 设备工厂方法 `Device::create()` 和 `Device::get_multi_device()` 静态方法。
- `src/bvh/multi.h` — `BVHMulti` 类，多设备 BVH 构建的核心数据结构。
- `src/scene/geometry.h` — `Geometry` 类，BVH 构建过程中涉及的几何体 BVH 所有权管理。
- `src/device/dummy/device.cpp` — 虚拟占位设备，与 MultiDevice 同为特殊用途的 `Device` 子类。
- `src/device/queue.h` — 设备队列定义，子设备的内核执行通过队列完成。
