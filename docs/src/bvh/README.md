# BVH (层次包围体) - 加速结构构建与管理

## 概述

`src/bvh/` 模块是 Cycles 路径追踪渲染器的**场景端 BVH 加速结构**实现。该模块负责在主机端（CPU）构建层次包围体（Bounding Volume Hierarchy），用于加速光线与场景几何体的求交测试。模块支持多种 BVH 后端，包括 Cycles 内置的 BVH2 二叉树、Intel Embree、NVIDIA OptiX、AMD HIP RT 以及 Apple Metal 等硬件加速方案。

BVH 的核心思想是将场景中的图元（三角形、曲线、点云等）按空间位置递归划分为层次化的包围盒树，使得光线遍历时能够快速跳过大量不相交的区域，将求交复杂度从 O(n) 降低到接近 O(log n)。

本模块属于 Cycles 场景构建管线的一部分，在渲染开始前生成加速结构数据，随后将数据打包传输至渲染设备（CPU/GPU）供内核侧遍历使用。

## 目录结构

| 文件名 | 类型 | 说明 |
|--------|------|------|
| `bvh.h` / `bvh.cpp` | 基类 | BVH 抽象基类与 `PackedBVH` 打包数据结构，工厂方法 `BVH::create()` |
| `bvh2.h` / `bvh2.cpp` | 后端 | BVH2 二叉树实现，包含构建、打包、重建（refit）逻辑 |
| `build.h` / `build.cpp` | 构建器 | `BVHBuild` 构建器，递归构建 BVH 树节点，支持多线程 |
| `node.h` / `node.cpp` | 节点 | BVH 树节点定义：`BVHNode`（抽象）、`InnerNode`（内部节点）、`LeafNode`（叶节点）|
| `params.h` | 参数 | `BVHParams` 参数类、`BVHReference`（图元引用）、`BVHRange`（构建范围）、`BVHSpatialStorage` |
| `binning.h` / `binning.cpp` | 算法 | `BVHObjectBinning` 基于分箱的 SAH 启发式对象划分 |
| `split.h` / `split.cpp` | 算法 | `BVHObjectSplit`（对象划分）、`BVHSpatialSplit`（空间划分）、`BVHMixedSplit`（混合划分）|
| `sort.h` / `sort.cpp` | 算法 | BVH 引用排序工具函数 `bvh_reference_sort()` |
| `unaligned.h` / `unaligned.cpp` | 算法 | `BVHUnaligned` 非轴对齐包围盒计算，用于曲线等非轴对齐图元 |
| `octree.h` / `octree.cpp` | 数据结构 | 体积八叉树 `Octree`，根据体积密度自适应细分，确定体积渲染步长 |
| `embree.h` / `embree.cpp` | 后端 | `BVHEmbree` Intel Embree 4 集成，支持 CPU 及 SYCL GPU 加速 |
| `optix.h` / `optix.cpp` | 后端 | `BVHOptiX` NVIDIA OptiX 硬件光线追踪集成 |
| `hiprt.h` / `hiprt.cpp` | 后端 | `BVHHIPRT` AMD HIP RT 硬件光线追踪集成 |
| `metal.h` / `metal.mm` | 后端 | Apple Metal 光线追踪加速结构集成 |
| `multi.h` / `multi.cpp` | 后端 | `BVHMulti` 多设备 BVH 容器，管理多个子 BVH |
| `CMakeLists.txt` | 构建 | CMake 构建配置，链接 `cycles_scene` 和 `cycles_util` |

## 核心类与数据结构

### BVH 类层次

```
BVH (抽象基类)
├── BVH2          — Cycles 内置二叉 BVH（软件遍历）
├── BVHEmbree     — Intel Embree 后端
├── BVHOptiX      — NVIDIA OptiX 后端
├── BVHHIPRT      — AMD HIP RT 后端
├── BVHMulti      — 多设备 BVH 容器
└── (Metal BVH)   — Apple Metal 后端（通过工厂函数创建）
```

### PackedBVH

`PackedBVH` 是 BVH 树的扁平化表示，用于传输至渲染设备：

- **`nodes`** (`array<int4>`)：内部节点数组，每个节点 4 个 `int4`，包含两个子节点的包围盒与子节点索引
- **`leaf_nodes`** (`array<int4>`)：叶节点数组
- **`object_node`** (`array<int>`)：对象索引到 BVH 节点索引的映射（实例化）
- **`prim_type`** / **`prim_index`** / **`prim_object`**：图元类型、索引和所属对象的映射
- **`prim_time`** (`array<float2>`)：运动模糊图元的时间范围
- **`root_index`**：根节点索引

### BVHNode 树节点

- **`BVHNode`**：抽象节点基类，包含 `bounds`（包围盒）、`visibility`（可见性标志）、可选的 `aligned_space`（非轴对齐变换）
- **`InnerNode`**：内部节点，最多支持 8 个子节点（`kNumMaxChildren = 8`），常用为 2 个子节点
- **`LeafNode`**：叶节点，存储图元范围 `[lo, hi)`

### BVHParams 构建参数

- **SAH 代价**：`sah_node_cost`、`sah_primitive_cost`（默认均为 1.0）
- **空间划分**：`use_spatial_split`（默认开启）、`spatial_split_alpha` 阈值
- **叶节点大小**：不同图元类型的最大叶节点大小（三角形 8、曲线 1、点云 8）
- **BVH 布局**：`bvh_layout`（默认 `BVH_LAYOUT_BVH2`）
- **深度限制**：`MAX_DEPTH = 64`、`MAX_SPATIAL_DEPTH = 48`

### Octree (体积八叉树)

- **`OctreeNode`**：八叉树节点，存储包围盒、深度、体积密度极值
- **`OctreeInternalNode`**：内部节点，包含 8 个子节点
- **`Octree`**：八叉树管理类，根据体积密度差异自适应细分，支持 OpenVDB 网格、并行构建

## 模块架构

```
场景数据 (Geometry, Object)
        │
        ▼
   BVHBuild (构建器)
   ├── BVHObjectBinning (分箱 SAH)
   ├── BVHObjectSplit / BVHSpatialSplit / BVHMixedSplit (划分策略)
   ├── BVHUnaligned (非轴对齐支持)
   └── bvh_reference_sort (排序)
        │
        ▼
   BVHNode 树 (InnerNode / LeafNode)
        │
        ▼
   BVH2::pack_nodes() ──→ PackedBVH ──→ 设备内存
        │
        ├── BVHEmbree::build()  ──→ RTCScene (Embree)
        ├── BVHOptiX            ──→ traversable_handle (OptiX)
        ├── BVHHIPRT::build()   ──→ hiprtGeometry (HIP RT)
        └── Metal BVH           ──→ Metal 加速结构
```

构建流程：
1. **收集引用**：从场景几何体（Mesh、Hair、PointCloud）收集图元引用 (`BVHReference`)
2. **递归划分**：使用 SAH 启发式在对象划分和空间划分之间选择最优策略
3. **构建树**：生成 `BVHNode` 树，支持多线程并行（`TaskPool`）
4. **打包**：将树扁平化为 `PackedBVH` 数组格式
5. **设备传输**：将打包数据上传至渲染设备

## 依赖关系

### 上游依赖（本模块依赖）

| 模块 | 说明 |
|------|------|
| `util/` | 基础类型（`BoundBox`、`Transform`、`array`、`vector`）、线程工具（`TaskPool`、`thread_mutex`）|
| `kernel/types.h` | `KernelBVHLayout` 枚举定义 |
| `device/memory.h` | 设备内存管理（`device_only_memory`，用于 OptiX/HIP RT）|
| `scene/` (`cycles_scene`) | 场景数据（`Geometry`、`Object`、`Mesh`、`Hair`、`PointCloud`）|
| Embree 4 SDK | `RTCDevice`、`RTCScene`（条件编译 `WITH_EMBREE`）|
| OptiX SDK | OptiX 加速结构 API（条件编译 `WITH_OPTIX`）|
| HIP RT SDK | `hiprtGeometry` 等（条件编译 `WITH_HIPRT`）|
| OpenVDB | 体积网格数据（条件编译 `WITH_OPENVDB`，用于 Octree）|

### 下游依赖（依赖本模块）

| 模块 | 说明 |
|------|------|
| `src/kernel/bvh/` | 内核端 BVH 遍历代码，使用 `PackedBVH` 打包数据 |
| `src/scene/` | 场景管理器调用 `BVH::create()` 构建加速结构 |
| `src/device/` | 各设备后端将 BVH 数据上传到 GPU |
| `src/integrator/` | 积分器通过内核调用场景求交函数 |

## 关键算法与实现细节

### SAH (Surface Area Heuristic) 构建

BVH 构建使用表面积启发式（SAH）来选择最优划分平面。SAH 代价公式：

```
Cost = C_trav * P(hit_node) + C_isect * (P(hit_left) * N_left + P(hit_right) * N_right)
```

其中 `P(hit)` 通过子节点包围盒的表面积比例估算。模块实现了三种划分策略：

- **对象划分** (`BVHObjectSplit`)：沿某一坐标轴将图元按重心位置分为左右两组，使用扫描线算法在 O(n) 时间内计算最优划分
- **空间划分** (`BVHSpatialSplit`)：沿坐标轴以固定间隔（32 个分箱）进行空间划分，允许图元被同时分配到两个子节点（图元复制），适用于大三角形跨越多个空间区域的情况
- **混合划分** (`BVHMixedSplit`)：先尝试对象划分，若左右子节点包围盒重叠面积超过阈值，再尝试空间划分，选择 SAH 代价最低的方案

### 分箱优化 (Object Binning)

`BVHObjectBinning` 实现了高效的多线程分箱算法（最多 32 个分箱），避免对所有图元进行完整排序。该算法源自 Intel 的光线追踪内核设计。

### 非轴对齐 BVH (Unaligned BVH)

对于毛发曲线等非轴对齐图元，`BVHUnaligned` 计算最优的定向包围盒（OBB），通过旋转变换使包围盒体积最小化，显著减少曲线图元的包围盒浪费空间。

### 体积八叉树 (Volume Octree)

`Octree` 用于体积渲染的自适应步长控制。其工作原理：
1. 将体积对象的包围盒划分为规则网格
2. 对网格中的每个体素采样体积密度
3. 根据密度差异递归细分八叉树节点（密度变化大的区域细分更深）
4. 将八叉树扁平化为数组并上传至内核，内核根据节点密度动态调整步长

### 运动模糊支持

通过 `prim_time` 存储每个图元的时间范围 `[time_from, time_to]`，支持运动三角形、运动曲线和运动点云。在构建时可将时间范围分割为多个步骤，每步创建独立的叶节点，以空间换取运动模糊渲染性能。

### 多后端架构

`BVH::create()` 工厂方法根据 `BVHLayout` 参数选择合适的后端：
- `BVH_LAYOUT_BVH2`：内置软件 BVH2
- `BVH_LAYOUT_EMBREE`：Intel Embree（CPU 和 oneAPI GPU）
- `BVH_LAYOUT_OPTIX`：NVIDIA OptiX（RTX 硬件加速）
- `BVH_LAYOUT_HIPRT`：AMD HIP RT
- `BVH_LAYOUT_METAL`：Apple Metal
- `BVH_LAYOUT_MULTI_*`：多设备组合（`BVHMulti` 容器）

## 参见

- `src/kernel/bvh/` — 内核端 BVH 遍历实现（GPU 兼容的遍历代码）
- `src/scene/object.cpp` / `src/scene/geometry.cpp` — 场景几何体管理与 BVH 触发
- `src/device/` — 各设备后端的 BVH 数据上传
- `src/kernel/geom/` — 内核端图元求交函数（三角形、曲线、点云）
- NVIDIA: "Understanding the Efficiency of Ray Traversal on GPUs" — BVH 遍历算法的理论基础
