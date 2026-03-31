# kernel/bvh - 内核端 BVH 遍历

## 概述

`src/kernel/bvh/` 模块实现了 Cycles 路径追踪渲染器在**内核端**（GPU/CPU 渲染设备上运行）的 BVH（层次包围体）遍历逻辑。该模块是光线-场景求交的核心执行代码，在渲染的每一条光线追踪中都会被调用。

本模块设计的核心特点是**通过宏模板生成多个特化版本**：针对不同图元类型组合（三角形、曲线、点云）和功能特征（运动模糊）分别编译优化版本，避免在运行时进行不必要的分支判断，从而最大化渲染性能。

模块提供了统一的场景求交入口函数（如 `scene_intersect()`），内部根据编译时宏定义和运行时设备类型，自动分派到对应的后端实现（内置 BVH2 遍历、Embree、OptiX、Metal RT 或 HIP RT）。

## 目录结构

| 文件名 | 类型 | 说明 |
|--------|------|------|
| `bvh.h` | 核心入口 | 场景求交统一入口，定义 `scene_intersect()`、`scene_intersect_local()`、`scene_intersect_shadow_all()`、`scene_intersect_volume()` 等函数，通过宏包含各遍历模板生成特化版本 |
| `types.h` | 类型定义 | BVH 遍历相关常量与宏：`BVH_STACK_SIZE`（192）、`ENTRYPOINT_SENTINEL`、特征位标志 `BVH_MOTION`/`BVH_HAIR`/`BVH_POINTCLOUD`、函数名拼接宏 |
| `traversal.h` | 遍历模板 | 常规光线遍历模板，支持三角形、曲线（毛发）、点云和运动模糊图元的求交，找到最近交点 |
| `local.h` | 遍历模板 | 局部遍历模板，用于次表面散射（SSS）、AO 和 Bevel 等需要在单个对象内寻找多个交点的场景 |
| `shadow_all.h` | 遍历模板 | 阴影光线遍历模板，记录所有透明交点，支持透明阴影和曲线阴影透明度 |
| `volume.h` | 遍历模板 | 体积遍历模板，查找最近的体积边界交点（仅处理 `SD_OBJECT_HAS_VOLUME` 对象）|
| `volume_all.h` | 遍历模板 | 体积遍历模板（多交点版本），记录所有体积边界交点，用于体积栈初始化和更新 |
| `nodes.h` | 节点求交 | BVH 节点求交函数：`bvh_aligned_node_intersect()`（轴对齐 AABB）、`bvh_unaligned_node_intersect()`（非轴对齐 OBB）、`bvh_node_intersect()`（统一分派）|
| `util.h` | 工具函数 | 光线有效性检查、自相交避免（`ray_offset()`）、交点排序、着色器标志获取、阴影链接过滤等实用函数 |

## 核心类与数据结构

### 场景求交入口函数

| 函数 | 说明 |
|------|------|
| `scene_intersect()` | 主求交函数，查找光线最近交点 |
| `scene_intersect_shadow()` | 阴影射线求交（快速路径，找到任意交点即返回）|
| `scene_intersect_local()` | 局部求交（单对象内部），用于 SSS/AO/Bevel |
| `scene_intersect_shadow_all()` | 透明阴影求交，记录所有交点并累计透明度 |
| `scene_intersect_volume()` | 体积求交，查找最近/所有体积边界交点 |

### BVH 遍历栈

- **栈大小**：`BVH_STACK_SIZE = 192`（64 对象 BVH + 64 网格 BVH + 64 节点分裂）
- **哨兵值**：`ENTRYPOINT_SENTINEL = 0x76543210`，标记栈底/遍历结束
- 遍历栈存储在 CUDA 线程局部内存（寄存器/本地内存）中

### BVH 特征标志

通过编译时宏 `BVH_FUNCTION_FEATURES` 组合以下标志：

| 标志 | 值 | 说明 |
|------|----|------|
| `BVH_MOTION` | 1 | 启用运动模糊图元支持 |
| `BVH_HAIR` | 2 | 启用毛发曲线图元支持（使用非轴对齐节点求交）|
| `BVH_POINTCLOUD` | 4 | 启用点云图元支持 |

### 节点求交函数

- **`bvh_aligned_node_intersect()`**：轴对齐 AABB 射线-节点求交，使用经典的 slab 方法，同时测试两个子节点
- **`bvh_unaligned_node_intersect()`**：非轴对齐 OBB 射线-节点求交，需要额外的坐标变换
- **`bvh_node_intersect()`**：统一入口，根据节点标志 `PATH_RAY_NODE_UNALIGNED` 分派到对齐或非对齐版本

## 模块架构

```
scene_intersect() / scene_intersect_shadow_all() / ...
        │
        ├── [Embree 后端]  kernel_embree_intersect()        (kernel/device/cpu/bvh.h)
        ├── [OptiX 后端]   optixTrace()                     (kernel/device/optix/bvh.h)
        ├── [Metal 后端]   metal RT intersector              (kernel/device/metal/bvh.h)
        ├── [HIP RT 后端]  hiprt intersect                  (kernel/device/hiprt/bvh.h)
        │
        └── [内置 BVH2]    bvh_intersect[_hair][_motion]()
                │
                ├── 内部节点遍历
                │   └── bvh_[aligned|unaligned]_node_intersect()  (nodes.h)
                │
                ├── 叶节点图元求交
                │   ├── triangle_intersect()        (kernel/geom/)
                │   ├── curve_intersect()            (kernel/geom/)
                │   ├── point_intersect()            (kernel/geom/)
                │   └── motion_triangle_intersect()  (kernel/geom/)
                │
                └── 实例化 push/pop
                    └── bvh_instance_[motion_]push/pop()
```

### 宏模板特化机制

`bvh.h` 通过多次 `#include` 遍历模板头文件并在每次包含前定义不同的宏来生成多个特化函数：

```c
// 生成 bvh_intersect() — 仅点云
#define BVH_FUNCTION_NAME bvh_intersect
#define BVH_FUNCTION_FEATURES BVH_POINTCLOUD
#include "kernel/bvh/traversal.h"

// 生成 bvh_intersect_hair() — 曲线 + 点云
#define BVH_FUNCTION_NAME bvh_intersect_hair
#define BVH_FUNCTION_FEATURES BVH_HAIR | BVH_POINTCLOUD
#include "kernel/bvh/traversal.h"

// 生成 bvh_intersect_motion() — 运动模糊 + 点云
// 生成 bvh_intersect_hair_motion() — 曲线 + 运动模糊 + 点云
```

运行时通过 `kernel_data.bvh.have_curves`、`kernel_data.bvh.have_motion` 等标志选择对应的特化版本。

## 内核函数入口

### 主要入口点

内核端的 BVH 遍历由积分器内核调用，主要入口点如下：

| 入口函数 | 调用位置 | 说明 |
|----------|----------|------|
| `scene_intersect()` | `kernel/integrator/shade_surface.h`、路径追踪主循环 | 每条光线的主求交调用 |
| `scene_intersect_shadow()` | `kernel/integrator/shade_shadow.h` | 不透明阴影射线快速测试 |
| `scene_intersect_local()` | `kernel/integrator/subsurface.h` | SSS 随机游走、Bevel 等局部求交 |
| `scene_intersect_shadow_all()` | `kernel/integrator/shade_shadow.h` | 透明阴影，记录所有半透明交点 |
| `scene_intersect_volume()` | `kernel/integrator/shade_volume.h` | 体积栈更新与体积边界检测 |

### 设备后端分派

在 `bvh.h` 中，编译时宏决定可用的后端：

```c
#if defined(__EMBREE__)      → kernel/device/cpu/bvh.h    // CPU Embree
#elif defined(__METALRT__)   → kernel/device/metal/bvh.h  // Apple Metal RT
#elif defined(__KERNEL_OPTIX__) → kernel/device/optix/bvh.h  // NVIDIA OptiX
#elif defined(__HIPRT__)     → kernel/device/hiprt/bvh.h  // AMD HIP RT
#else                        → 内置 BVH2 遍历             // 软件回退
```

对于 oneAPI + Embree GPU 的情况，使用 SYCL 特化常量 (`specialization_id<RTCFeatureFlags>`) 在运行时选择是否使用 Embree GPU 或回退到 BVH2。

## GPU 兼容性

### 设备支持矩阵

| 设备 | 后端 | 硬件加速 | 说明 |
|------|------|----------|------|
| NVIDIA GPU (RTX) | OptiX | RT Core 硬件加速 | 通过 `optixTrace()` 调用 |
| AMD GPU | HIP RT | RDNA2+ 硬件加速 | 通过 HIP RT API 调用 |
| Apple GPU | Metal RT | Apple Silicon 硬件加速 | Metal Ray Tracing API |
| Intel GPU | oneAPI + Embree | Xe HPG 硬件加速 | Embree SYCL 后端 |
| CPU | Embree 或 BVH2 | 无（软件遍历）| Embree 利用 SIMD 优化 |

### GPU 内核注意事项

- **内联策略**：GPU 上求交函数使用 `ccl_device_forceinline`，CPU 上使用 `ccl_device_inline`（`types.h` 中定义）
- **栈内存**：遍历栈分配在线程局部内存中，GPU 上对应 CUDA local memory / 寄存器
- **分支优化**：通过编译时特化避免运行时分支，不同图元组合生成独立函数
- **可见性标志**：通过 `__VISIBILITY_FLAG__` 宏控制是否在节点求交时检查可见性（约 5% 性能影响）
- **自相交避免**：`ray_offset()` 使用整数位操作实现鲁棒的浮点偏移（来自 Ray Tracing Gems 第 6 章）
- **阴影链接**：通过 `__SHADOW_LINKING__` 宏支持选择性阴影投射（Blender 4.0+）

### 编译宏依赖

| 宏 | 说明 |
|----|------|
| `__KERNEL_GPU__` | GPU 编译路径 |
| `__KERNEL_CUDA__` | CUDA 编译路径 |
| `__KERNEL_OPTIX__` | OptiX 编译路径 |
| `__KERNEL_ONEAPI__` | Intel oneAPI 编译路径 |
| `__EMBREE__` | Embree 可用 |
| `__METALRT__` | Metal RT 可用 |
| `__HIPRT__` | HIP RT 可用 |
| `__BVH2__` | 启用内置 BVH2 遍历（无硬件加速时的回退）|
| `__HAIR__` | 启用毛发曲线支持 |
| `__POINTCLOUD__` | 启用点云支持 |
| `__OBJECT_MOTION__` | 启用运动模糊 |
| `__VOLUME__` | 启用体积渲染 |
| `__SHADOW_RECORD_ALL__` | 启用透明阴影记录 |
| `__BVH_LOCAL__` | 启用局部 BVH 遍历（SSS/Bevel）|
| `__SHADOW_LINKING__` | 启用阴影链接 |

## 依赖关系

### 上游依赖（本模块依赖）

| 模块 | 说明 |
|------|------|
| `kernel/globals.h` | 全局内核数据访问宏 `kernel_data_fetch()` |
| `kernel/types.h` | 内核类型定义（`Ray`、`Intersection`、`LocalIntersection` 等）|
| `kernel/integrator/state.h` | 积分器状态（用于阴影交点记录）|
| `kernel/geom/` | 图元求交函数（`triangle_intersect`、`curve_intersect`、`point_intersect`、`motion_triangle_intersect`）|
| `kernel/geom/object.h` | 对象变换（实例化 push/pop）|
| `kernel/device/*/bvh.h` | 各设备后端的 BVH 求交实现 |

### 下游依赖（依赖本模块）

| 模块 | 说明 |
|------|------|
| `kernel/integrator/` | 积分器内核调用场景求交入口函数 |
| `kernel/integrator/shade_surface.h` | 路径追踪主循环中的光线求交 |
| `kernel/integrator/subsurface.h` | 次表面散射局部求交 |
| `kernel/integrator/shade_shadow.h` | 阴影射线求交 |
| `kernel/integrator/shade_volume.h` | 体积边界求交 |

## 关键算法与实现细节

### BVH2 遍历算法

内置 BVH2 遍历基于 "Understanding the Efficiency of Ray Traversal on GPUs" (Aila & Laine, 2009)，采用经典的栈式深度优先遍历：

1. 从根节点开始，将 `ENTRYPOINT_SENTINEL` 压入栈底
2. **内部节点处理**：测试光线与两个子节点的包围盒求交
   - 两个都相交：将较远子节点入栈，继续遍历较近子节点
   - 仅一个相交：继续遍历该子节点
   - 都不相交：从栈中弹出下一个节点
3. **叶节点处理**：遍历叶节点中的所有图元，调用对应的图元求交函数
4. **实例化处理**：遇到实例时执行 `bvh_instance_push()`（变换光线到对象空间），遍历完实例后执行 `bvh_instance_pop()`（恢复光线）
5. **终止条件**：栈为空时遍历结束

### 透明阴影遍历

`shadow_all.h` 实现了一种特殊的遍历策略：
- 记录所有与光线相交的半透明表面（最多 `INTEGRATOR_SHADOW_ISECT_SIZE` 个）
- 对于不透明表面立即返回 `true`（完全遮挡）
- 对于曲线，使用烘焙的阴影透明度衰减 (`intersection_curve_shadow_transparency()`)
- 超过最大透明弹跳次数时视为完全遮挡
- 使用替换策略：当记录满时，替换最远的交点

### 自相交避免

`util.h` 中的 `ray_offset()` 函数使用整数位操作实现精确的浮点偏移，来自 Ray Tracing Gems 第 6 章 "A Fast and Robust Method for Avoiding Self-Intersection"。该方法在所有浮点精度范围内都能正确工作。

## 参见

- `src/bvh/` — 主机端 BVH 构建模块
- `src/kernel/geom/` — 内核端几何图元求交实现
- `src/kernel/device/cpu/bvh.h` — Embree CPU 后端
- `src/kernel/device/optix/bvh.h` — OptiX 后端
- `src/kernel/device/metal/bvh.h` — Metal RT 后端
- `src/kernel/device/hiprt/bvh.h` — HIP RT 后端
- `src/kernel/integrator/` — 积分器内核（调用方）
- Aila & Laine, "Understanding the Efficiency of Ray Traversal on GPUs", HPG 2009
- Woop et al., "A Fast and Robust Method for Avoiding Self-Intersection", Ray Tracing Gems Ch.6
