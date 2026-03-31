# devicescene.h / devicescene.cpp - 设备端场景数据容器

## 概述

本文件定义了 `DeviceScene` 类，它是 Cycles 渲染器中所有设备端（GPU/CPU 渲染内核）场景数据的统一容器。包含 BVH 加速结构、网格、曲线、点云、物体、属性、灯光、灯光树、粒子、着色器、查找表、采样模式、IES 灯光和体积八叉树等数据的设备向量（`device_vector`），以及核心的 `KernelData` 结构体。所有设备向量在构造时绑定到指定设备并使用全局内存类型。

## 类与结构体

### DeviceScene
- **功能**: 设备端场景数据的统一容器，持有所有需要上传到渲染设备的数据缓冲区
- **BVH 数据**:
  - `bvh_nodes` — BVH 内部节点 (`int4`)
  - `bvh_leaf_nodes` — BVH 叶节点 (`int4`)
  - `object_node` — 物体到 BVH 节点的映射 (`int`)
  - `prim_type` — 图元类型 (`int`)
  - `prim_visibility` — 图元可见性标志 (`uint`)
  - `prim_index` — 图元索引 (`int`)
  - `prim_object` — 图元所属物体 (`int`)
  - `prim_time` — 图元运动模糊时间范围 (`float2`)
- **网格数据**:
  - `tri_verts` — 三角形顶点坐标 (`packed_float3`)
  - `tri_shader` — 三角形着色器索引 (`uint`)
  - `tri_vnormal` — 三角形顶点法线 (`packed_float3`)
  - `tri_vindex` — 三角形顶点索引 (`packed_uint3`)
- **曲线数据**:
  - `curves` — 曲线描述 (`KernelCurve`)
  - `curve_keys` — 曲线控制点 (`float4`)
  - `curve_segments` — 曲线段 (`KernelCurveSegment`)
- **点云数据**:
  - `points` — 点位置和半径 (`float4`)
  - `points_shader` — 点着色器索引 (`uint`)
- **物体数据**:
  - `objects` — 物体描述 (`KernelObject`)
  - `object_motion_pass` — 运动渲染通道变换 (`Transform`)
  - `object_motion` — 运动模糊变换 (`DecomposedTransform`)
  - `object_flag` — 物体标志 (`uint`)
  - `object_prim_offset` — 物体图元偏移 (`uint`)
- **相机数据**:
  - `camera_motion` — 相机运动模糊变换 (`DecomposedTransform`)
- **属性数据**:
  - `attributes_map` — 属性映射表 (`AttributeMap`)
  - `attributes_float/float2/float3/float4/uchar4` — 不同类型的属性数据
- **灯光数据**:
  - `light_distribution` — 灯光分布 (`KernelLightDistribution`)
  - `lights` — 灯光描述 (`KernelLight`)
  - `light_background_marginal_cdf` / `light_background_conditional_cdf` — 背景灯光 CDF
- **灯光树数据**:
  - `light_tree_nodes` — 灯光树节点 (`KernelLightTreeNode`)
  - `light_tree_emitters` — 灯光树发射器 (`KernelLightTreeEmitter`)
  - `light_to_tree`, `object_to_tree`, `object_lookup_offset`, `triangle_to_tree` — 映射表
- **粒子数据**: `particles` — 粒子描述 (`KernelParticle`)
- **着色器数据**:
  - `svm_nodes` — SVM 着色器虚拟机节点 (`int4`)
  - `shaders` — 着色器描述 (`KernelShader`)
- **查找表**: `lookup_table` — 通用查找表 (`float`)
- **积分器**: `sample_pattern_lut` — 采样模式查找表 (`float`)
- **IES 灯光**: `ies_lights` — IES 灯光数据 (`float`)
- **体积数据**:
  - `volume_tree_nodes` — 体积八叉树节点 (`KernelOctreeNode`)
  - `volume_tree_roots` — 体积八叉树根 (`KernelOctreeRoot`)
  - `volume_tree_root_ids` — 体积八叉树根 ID (`int`)
  - `volume_step_size` — 体积步进大小 (`float`)
- **核心数据**: `data` — `KernelData` 结构体，包含所有内核常量参数

## 核心函数

构造函数 `DeviceScene(Device *device)` 是唯一的实现，使用初始化列表将所有 `device_vector` 绑定到指定设备，统一使用 `MEM_GLOBAL` 内存类型。构造后将 `data` 清零初始化。

## 依赖关系

- **内部头文件**: `kernel/types.h`（内核数据类型定义）, `device/device.h`, `device/memory.h`
- **cpp 额外引用**: `device/device.h`, `device/memory.h`
- **被引用**: `scene/scene.h`（Scene 持有 `DeviceScene dscene` 成员）, `scene/scene.cpp`

## 实现细节 / 关键算法

1. **全局内存**: 所有设备向量使用 `MEM_GLOBAL` 内存类型，表示该数据在整个渲染过程中持续存在于设备端，可被所有渲染内核全局访问。
2. **零初始化**: 构造函数通过 `memset(&data, 0, sizeof(data))` 将 `KernelData` 结构体清零，确保所有内核常量有确定的初始值。
3. **数据布局**: 数据按功能分组（BVH、网格、曲线、点云、物体、属性、灯光、着色器等），每组包含多个设备向量。这种布局匹配内核端的数据访问模式。
4. **被动容器**: DeviceScene 本身不包含任何数据更新逻辑，仅作为数据容器。各管理器（GeometryManager、LightManager 等）负责写入对应的设备向量。

## 关联文件

- `scene/scene.h/.cpp` — 场景管理（持有 `DeviceScene dscene`，各管理器通过 `&dscene` 访问设备数据）
- `kernel/types.h` — 内核类型定义（`KernelData`, `KernelObject`, `KernelLight` 等）
- `device/memory.h` — 设备内存（`device_vector`, `device_texture` 定义）
- `device/device.h` — 设备接口
