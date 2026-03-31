# data_arrays.h - 内核全局数据数组声明注册表

## 概述

`data_arrays.h` 是 Cycles 渲染内核中所有全局设备端数据数组的集中声明文件。它通过 `KERNEL_DATA_ARRAY(type, name)` 宏以声明式方式定义了渲染管线所需的全部数据缓冲区，涵盖层次包围体(BVH)节点、几何体、光源、着色器、纹理等核心数据。该文件本身不包含任何实现逻辑，而是作为"X-Macro"模板被多次 `#include`，由各个设备后端根据自身需求展开为对应的数据结构声明、内存分配或绑定代码。

## 类与结构体

本文件不定义类或结构体，但声明了引用以下结构体类型的数组：

| 数组名 | 类型 | 用途 |
|---|---|---|
| `bvh_nodes` | `float4` | BVH2 内部节点数据（非 OptiX/Embree 时使用） |
| `bvh_leaf_nodes` | `float4` | BVH2 叶子节点数据 |
| `prim_type` | `uint` | 图元类型标识 |
| `prim_visibility` | `uint` | 图元可见性标志 |
| `prim_index` | `uint` | 图元索引 |
| `prim_object` | `uint` | 图元所属对象索引 |
| `object_node` | `uint` | 对象对应的 BVH 节点 |
| `prim_time` | `float2` | 图元运动模糊时间范围 |
| `objects` | `KernelObject` | 场景对象数据 |
| `object_motion_pass` | `Transform` | 对象运动通道变换 |
| `object_motion` | `DecomposedTransform` | 对象运动模糊变换序列 |
| `object_flag` | `uint` | 对象标志位 |
| `object_prim_offset` | `uint` | 对象图元偏移 |
| `camera_motion` | `DecomposedTransform` | 摄像机运动模糊变换 |
| `tri_shader` | `uint` | 三角形着色器索引 |
| `tri_vnormal` | `packed_float3` | 三角形顶点法线 |
| `tri_vindex` | `packed_uint3` | 三角形顶点索引 |
| `tri_verts` | `packed_float3` | 三角形顶点坐标 |
| `curves` | `KernelCurve` | 曲线（毛发）数据 |
| `curve_keys` | `float4` | 曲线控制点 |
| `curve_segments` | `KernelCurveSegment` | 曲线段数据 |
| `points` | `float4` | 点云数据 |
| `points_shader` | `uint` | 点云着色器索引 |
| `attributes_map` | `AttributeMap` | 属性映射表 |
| `attributes_float` | `float` | 浮点属性数据 |
| `attributes_float2` | `float2` | 二维浮点属性 |
| `attributes_float3` | `packed_float3` | 三维浮点属性 |
| `attributes_float4` | `float4` | 四维浮点属性 |
| `attributes_uchar4` | `uchar4` | 字节属性数据 |
| `light_distribution` | `KernelLightDistribution` | 光源分布数据 |
| `lights` | `KernelLight` | 光源参数 |
| `light_background_marginal_cdf` | `float2` | 环境光边缘 CDF |
| `light_background_conditional_cdf` | `float2` | 环境光条件 CDF |
| `light_tree_nodes` | `KernelLightTreeNode` | 光源树节点 |
| `light_tree_emitters` | `KernelLightTreeEmitter` | 光源树发射体 |
| `light_to_tree` | `uint` | 光源到光源树的映射 |
| `object_to_tree` | `uint` | 对象到光源树的映射 |
| `object_lookup_offset` | `uint` | 对象查找偏移 |
| `triangle_to_tree` | `uint` | 三角形到光源树的映射 |
| `particles` | `KernelParticle` | 粒子数据 |
| `svm_nodes` | `uint4` | 着色器虚拟机(SVM)指令节点 |
| `shaders` | `KernelShader` | 着色器描述数据 |
| `lookup_table` | `float` | 通用查找表 |
| `sample_pattern_lut` | `float` | 表格化 Sobol 采样模式 |
| `texture_info` | `TextureInfo` | 图像纹理信息 |
| `ies` | `float` | IES 光域数据 |
| `volume_tree_nodes` | `KernelOctreeNode` | 体积八叉树节点 |
| `volume_tree_roots` | `KernelOctreeRoot` | 体积八叉树根节点 |
| `volume_tree_root_ids` | `int` | 体积八叉树根节点 ID |
| `volume_step_size` | `float` | 体积步长大小 |

## 枚举与常量

无独立枚举或常量定义。文件头部定义了 `KERNEL_DATA_ARRAY` 宏的默认空实现（当调用方未预先定义时），并在文件末尾使用 `#undef` 清理。

## 核心函数

无函数定义。本文件是纯粹的宏驱动声明模板。

## 依赖关系

- **内部头文件**: `kernel/types.h`（在文件开头直接 include，提供所有结构体类型定义）
- **被引用**:
  - 各设备后端全局变量定义: `kernel/device/cpu/globals.h`, `kernel/device/cuda/globals.h`, `kernel/device/hip/globals.h`, `kernel/device/hiprt/globals.h`, `kernel/device/metal/globals.h`, `kernel/device/oneapi/globals.h`, `kernel/device/optix/globals.h`
  - CPU 内核实现: `kernel/device/cpu/kernel.cpp`
  - 各设备后端内存管理: `device/cpu/device_impl.cpp`, `device/cuda/device_impl.cpp`, `device/hip/device_impl.cpp`, `device/hiprt/device_impl.cpp`, `device/metal/device_impl.mm`, `device/oneapi/device_impl.cpp`, `device/optix/device_impl.cpp`

## 实现细节 / 关键算法

本文件采用经典的 **X-Macro 模式**（又称"include 模板"模式）。调用方在 `#include` 本文件之前先定义 `KERNEL_DATA_ARRAY(type, name)` 宏，将其展开为所需的代码（例如声明指针成员、分配显存、绑定纹理等），随后本文件会对所有数据数组逐一调用该宏。这种模式确保了数据数组列表在整个代码库中保持唯一的"真相源"（single source of truth），避免了在不同设备后端之间手工同步数组定义的风险。

数据数组按功能模块分组排列：BVH -> 对象 -> 摄像机 -> 三角形 -> 曲线 -> 点云 -> 属性 -> 光源 -> 光源树 -> 粒子 -> 着色器 -> 查找表 -> 采样模式 -> 纹理 -> IES -> 体积。

## 关联文件

- `kernel/types.h` — 所有 `Kernel*` 结构体的定义
- `kernel/data_template.h` — 类似的 X-Macro 模板，用于内核常量数据结构
- `kernel/globals.h` — CPU 端 KernelGlobals 定义入口
- `kernel/device/*/globals.h` — 各设备后端利用本文件展开数组声明
