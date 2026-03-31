# device/multi - 多设备聚合层

## 概述

`multi/` 子目录实现了 Cycles 的多设备聚合后端。`MultiDevice` 继承自 `Device` 基类，将多个物理设备（GPU+GPU 或 GPU+CPU）组合为一个逻辑设备，使上层渲染逻辑透明地使用多设备并行渲染。

核心特点：
- **透明聚合**：上层代码通过统一的 `Device` 接口操作 `MultiDevice`，无需感知底层子设备数量和类型。
- **对等岛（Peer Islands）**：将支持 P2P 内存访问的设备分组，同一岛内的设备共享内存分配，减少数据复制。
- **内存分发**：内存分配自动分发到所有对等岛，纹理和全局内存在岛内所有设备上同步。
- **BVH 广播**：将加速结构构建广播到所有子设备，每个设备独立持有自己的 BVH 副本。
- **硬件光线追踪一致性**：确保所有 GPU 子设备使用相同的硬件光线追踪设置（全启或全关）。
- **CPU 线程调控**：多设备包含 CPU 时，自动减少 CPU 线程数以为 GPU 让出资源。

## 目录结构

| 文件名 | 类型 | 说明 |
|--------|------|------|
| `device.h` | 头文件 | `device_multi_create()` 工厂函数 |
| `device.cpp` | 源文件 | `MultiDevice` 类完整实现（类定义和实现均在此文件） |

## 核心类与数据结构

### MultiDevice

定义位置：`device.cpp`

继承自 `Device`。关键成员：

| 成员 | 说明 |
|------|------|
| `devices` | `list<SubDevice>` — 子设备列表（链表，保证指针稳定性） |
| `unique_key` | 内存分配键的递增计数器 |
| `peer_islands` | `vector<vector<SubDevice*>>` — 对等岛列表，相互可 P2P 访问的设备分组 |

### SubDevice

`MultiDevice` 的内部结构体，表示一个子设备：

| 成员 | 说明 |
|------|------|
| `stats` | 独立的统计信息 |
| `device` | 实际的 `Device` 实例 |
| `ptr_map` | 逻辑内存键到子设备内存指针的映射 |
| `peer_island_index` | 所属的对等岛索引 |

### 关键方法

| 方法 | 说明 |
|------|------|
| `mem_alloc()` | 为每个对等岛中负载最低的设备分配内存 |
| `mem_copy_to()` | 将数据拷贝到所有对等岛（纹理/全局内存同步到岛内所有设备） |
| `mem_copy_from()` | 按子设备数量均分高度，从各设备拷贝渲染结果片段 |
| `mem_free()` | 释放所有对等岛上的分配 |
| `build_bvh()` | 共享 BVH（BVH2/Embree）由最后一个设备构建；多 BVH（Multi-OptiX/Metal/HIPRT）广播到所有设备 |
| `load_kernels()` | 在所有子设备上加载内核 |
| `foreach_device()` | 遍历所有子设备执行回调 |
| `get_bvh_layout_mask()` | 计算所有子设备支持的 BVH 布局交集，确定 Multi-BVH 类型 |
| `verify_hardware_raytracing()` | 确保所有 GPU 子设备硬件光线追踪设置一致 |

### 内存管理策略

1. **内存键映射**：`MultiDevice` 使用逻辑键（`unique_key`）作为 `mem.device_pointer`，每个子设备维护键到实际设备指针的映射 (`ptr_map`)。
2. **对等岛分配**：每个对等岛选择内存使用最低的设备进行实际分配，岛内其他设备通过 P2P 访问共享。
3. **纹理同步**：`MEM_GLOBAL` 和 `MEM_TEXTURE` 类型的内存在对等岛内所有设备上创建纹理对象。
4. **主机内存**：优先使用 GPU 设备的 `host_alloc()` 以获得映射内存优化。

## 硬件要求

- 需要两个或更多渲染设备（同类或异类）
- 支持的组合：
  - 多个同类 GPU（如多块 NVIDIA RTX GPU）
  - GPU + CPU 混合
  - 不同类型的 GPU（如 OptiX + CPU，会使用 Multi-BVH）

## API 封装

`MultiDevice` 本身不直接调用任何底层 GPU API，而是将所有操作代理到子设备的对应方法。

## 依赖关系

### 上游依赖（本模块依赖）

| 模块 | 用途 |
|------|------|
| `device/` | `Device` 基类、`DeviceQueue` 基类、内存体系 |
| `bvh/multi.h` | `BVHMulti` 多设备 BVH |
| `scene/geometry.h` | 几何数据（BVH 构建时的 BVH 指针交换） |
| `util/` | 列表、映射、日志 |

### 下游依赖（依赖本模块）

| 模块 | 用途 |
|------|------|
| `device/device.cpp` | 当 `DeviceInfo::multi_devices` 非空时创建 `MultiDevice` |
| `session/` | 通过 `Device::get_multi_device()` 组合多个设备信息 |

## 参见

- `src/device/device.h` — `Device` 基类
- `src/device/device.cpp` — `Device::get_multi_device()` 静态方法
- `src/bvh/multi.h` — 多设备 BVH 容器
