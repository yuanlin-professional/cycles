# geometry.h - 几何信息节点的着色器虚拟机实现

## 概述

本文件实现了着色器虚拟机（SVM）中一组用于获取场景几何信息的节点，包括几何节点（Geometry）、物体信息节点（Object Info）、粒子信息节点（Particle Info）、毛发信息节点（Hair Info）和点云信息节点（Point Info）。这些节点为着色器提供了当前着色点的各种几何属性数据，是程序化着色的基础输入源。

## 核心函数

### `svm_node_geometry`

```c
ccl_device_noinline void svm_node_geometry(
    KernelGlobals kg,
    ccl_private ShaderData *sd,
    ccl_private float *stack,
    const uint type,
    const uint out_offset)
```

**功能**：输出当前着色点的基本几何属性。

**支持的输出类型**：

| 枚举值 | 输出 | 说明 |
|--------|------|------|
| `NODE_GEOM_P` | 位置 | 着色点的世界空间位置 `sd->P` |
| `NODE_GEOM_N` | 法线 | 着色法线 `sd->N`（含平滑和凹凸） |
| `NODE_GEOM_T` | 切线 | 表面切线（需 `__DPDU__` 编译选项） |
| `NODE_GEOM_I` | 入射方向 | 入射光线方向 `sd->wi` |
| `NODE_GEOM_Ng` | 几何法线 | 真实几何法线 `sd->Ng`（无平滑） |
| `NODE_GEOM_uv` | 重心坐标 | 三角形重心坐标 `(1-u-v, u, 0)` |

### `svm_node_geometry_bump_dx` / `svm_node_geometry_bump_dy`

```c
ccl_device_noinline void svm_node_geometry_bump_dx(...)
ccl_device_noinline void svm_node_geometry_bump_dy(...)
```

**功能**：几何节点的凹凸贴图变体。对位置和 UV 输出进行微分偏移，用于凹凸法线的有限差分计算。

- **位置**：使用 `svm_node_bump_P_dx/dy` 返回微分偏移后的位置
- **UV**：对 u/v 添加微分偏移 `du.dx/dy * bump_filter_width`
- **其他属性**：直接委托给普通的 `svm_node_geometry`

### `svm_node_object_info`

```c
ccl_device_noinline void svm_node_object_info(
    KernelGlobals kg,
    ccl_private ShaderData *sd,
    ccl_private float *stack,
    const uint type,
    const uint out_offset)
```

**功能**：输出当前物体的元数据信息。

**支持的输出类型**：

| 枚举值 | 输出 | 类型 |
|--------|------|------|
| `NODE_INFO_OB_LOCATION` | 物体位置 | float3 |
| `NODE_INFO_OB_COLOR` | 物体颜色 | float3 |
| `NODE_INFO_OB_ALPHA` | 物体透明度 | float |
| `NODE_INFO_OB_INDEX` | 物体索引 | float |
| `NODE_INFO_MAT_INDEX` | 材质索引 | float |
| `NODE_INFO_OB_RANDOM` | 物体随机数 | float |

### `svm_node_particle_info`

```c
ccl_device_noinline void svm_node_particle_info(...)
```

**功能**：输出粒子系统相关信息（索引、随机值、年龄、生命周期、位置、大小、速度、角速度）。

### `svm_node_hair_info`

```c
ccl_device_noinline void svm_node_hair_info(...)
```

**功能**：输出毛发/曲线相关信息（是否为毛发、粗细、切线法线等）。需 `__HAIR__` 编译选项。

### `svm_node_point_info`

```c
ccl_device_noinline void svm_node_point_info(...)
```

**功能**：输出点云相关信息（位置、半径等）。需 `__POINTCLOUD__` 编译选项。

## 依赖关系

- **内部头文件**：`kernel/geom/curve.h`、`kernel/geom/primitive.h`、`kernel/svm/attribute.h`、`kernel/svm/util.h`、`util/hash.h`
- **被引用**：`kernel/svm/svm.h`（SVM 主调度器）

## 实现细节 / 关键算法

- **凹凸变体的设计模式**：`svm_node_geometry_bump_dx/dy` 是一种常见的 SVM 模式——对需要参与凹凸计算的节点提供带微分偏移的变体。只有位置和 UV 这类连续变化的属性需要特殊处理，其他属性（法线、入射方向等）不需要微分偏移。

- **粒子随机数生成**：粒子随机值通过 `hash_uint2_to_float(particle_index, 0)` 生成，使用粒子索引作为哈希种子，确保每个粒子获得确定性但看似随机的值。

- **毛发属性的延迟处理**：部分毛发信息（截距、长度、随机值）标记为"handled as attribute"，即通过属性系统而非此节点函数处理。

- **条件编译**：毛发和点云相关函数使用 `#ifdef __HAIR__` 和 `#ifdef __POINTCLOUD__` 守护，允许在不需要这些特性的内核变体中省略相关代码。

- **float3 vs float 输出**：物体信息节点中，位置和颜色使用 `stack_store_float3` 输出后直接返回，而标量值（透明度、索引等）使用 `stack_store_float` 输出。这种分支结构避免了不必要的类型转换。

## 关联文件

- `kernel/svm/svm.h`：SVM 主调度入口
- `kernel/svm/attribute.h`：属性节点（毛发属性的实际处理）
- `kernel/geom/curve.h`：毛发几何函数（`curve_thickness`、`curve_tangent_normal`）
- `kernel/geom/primitive.h`：图元几何函数（`primitive_tangent`）
- `util/hash.h`：哈希函数用于粒子随机数
