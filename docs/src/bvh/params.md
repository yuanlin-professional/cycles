# params.h - 层次包围体(BVH)参数、引用与空间分割存储定义

## 概述

本头文件定义了层次包围体(BVH)系统的核心参数和数据结构，是整个 BVH 子系统中被引用最广泛的文件。它包含 BVH 构建参数（`BVHParams`）、图元引用（`BVHReference`）、构建范围（`BVHRange`）、空间分割(Spatial Split)存储容器（`BVHSpatialBin`、`BVHSpatialStorage`）等基础类型，以及 BVH 布局和类型的枚举定义。

## 类与结构体

### `BVHLayout` (类型别名)
BVH 树的布局类型，别名为 `KernelBVHLayout`。定义了每个节点的子节点数量（如 BVH2 表示二叉树）。

### `BVHType` (枚举)
BVH 的更新策略类型：

| 值 | 说明 |
|----|------|
| `BVH_TYPE_DYNAMIC` | 支持动态更新几何体，适用于视口编辑时快速更新，但渲染较慢 |
| `BVH_TYPE_STATIC` | 针对特定场景计算，几何修改需完全重建，构建较慢但渲染性能最优 |

### `BVHLayoutMask` (类型别名)
`int` 类型的位标志，用于表示特定区域支持的 BVH 布局集合。

### `BVHParams`
BVH 构建的核心参数类。

**空间分割(Spatial Split)参数:**

| 成员 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `use_spatial_split` | `bool` | `true` | 是否启用空间分割(Spatial Split) |
| `spatial_split_alpha` | `float` | `1e-5f` | 空间分割(Spatial Split)面积阈值 |

**非对齐节点参数:**

| 成员 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `unaligned_split_threshold` | `float` | `0.7f` | 非对齐节点创建阈值 |

**表面积启发式(SAH)开销:**

| 成员 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `sah_node_cost` | `float` | `1.0f` | 节点遍历开销 |
| `sah_primitive_cost` | `float` | `1.0f` | 图元相交测试开销 |

**叶节点大小限制:**

| 成员 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `min_leaf_size` | `int` | `1` | 叶节点最少图元数 |
| `max_triangle_leaf_size` | `int` | `8` | 三角形叶节点最大图元数 |
| `max_motion_triangle_leaf_size` | `int` | `8` | 运动三角形叶节点最大图元数 |
| `max_curve_leaf_size` | `int` | `1` | 曲线叶节点最大图元数 |
| `max_motion_curve_leaf_size` | `int` | `4` | 运动曲线叶节点最大图元数 |
| `max_point_leaf_size` | `int` | `8` | 点云叶节点最大图元数 |
| `max_motion_point_leaf_size` | `int` | `8` | 运动点云叶节点最大图元数 |

**其他参数:**

| 成员 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `top_level` | `bool` | `false` | 是否为场景级别（顶层）BVH |
| `bvh_layout` | `BVHLayout` | `BVH_LAYOUT_BVH2` | 要构建的 BVH 布局 |
| `use_unaligned_nodes` | `bool` | `false` | 是否使用非对齐包围盒（仅用于曲线） |
| `use_compact_structure` | `bool` | `false` | 是否使用紧凑加速结构（Embree） |
| `num_motion_triangle_steps` | `int` | `0` | 运动三角形时间步数 |
| `num_motion_curve_steps` | `int` | `0` | 运动曲线时间步数 |
| `num_motion_point_steps` | `int` | `0` | 运动点云时间步数 |
| `bvh_type` | `int` | `0` | BVH 类型（对应 `SceneParams`） |
| `curve_subdivisions` | `int` | `4` | 曲线细分数（用于 Embree） |

**常量:**

| 常量 | 值 | 说明 |
|------|-----|------|
| `MAX_DEPTH` | `64` | BVH 树最大深度 |
| `MAX_SPATIAL_DEPTH` | `48` | 空间分割(Spatial Split)最大深度 |
| `NUM_SPATIAL_BINS` | `32` | 空间分割(Spatial Split)的 bin 数量 |

**关键方法:**

| 方法 | 说明 |
|------|------|
| `cost(num_nodes, num_primitives)` | 计算表面积启发式(SAH)总开销 |
| `primitive_cost(n)` | 计算 n 个图元的开销 |
| `node_cost(n)` | 计算 n 个节点的开销 |
| `small_enough_for_leaf(size, level)` | 判断是否满足叶节点条件 |
| `use_motion_steps()` | 判断是否使用运动步 |
| `best_bvh_layout(requested, supported)` | 获取设备支持的最佳 BVH 布局（静态方法） |

### `BVHReference`
图元引用类。通过将图元索引和对象索引编码进 `BoundBox` 的 `w` 分量中，减少内存占用并优化对齐。

| 方法 | 返回类型 | 说明 |
|------|----------|------|
| `bounds()` | `const BoundBox &` | 图元包围盒 |
| `prim_index()` | `int` | 图元索引（编码在 `rbounds.min.w` 中） |
| `prim_object()` | `int` | 对象索引（编码在 `rbounds.max.w` 中） |
| `prim_type()` | `int` | 图元类型 |
| `time_from()` / `time_to()` | `float` | 运动模糊时间范围 |

### `BVHRange`
构建范围类，表示引用数组中一个子集的位置和范围。同样使用 `BoundBox` 的 `w` 分量打包整数以优化内存对齐。

| 方法 | 返回类型 | 说明 |
|------|----------|------|
| `bounds()` | `const BoundBox &` | 范围的包围盒 |
| `cent_bounds()` | `const BoundBox &` | 质心包围盒 |
| `start()` | `int` | 起始索引 |
| `size()` | `int` | 引用数量 |
| `end()` | `int` | 结束索引（`start + size`） |

### `BVHSpatialBin`
空间分割(Spatial Split)使用的 bin 结构体。

| 成员 | 类型 | 说明 |
|------|------|------|
| `bounds` | `BoundBox` | bin 的包围盒 |
| `enter` | `int` | 进入此 bin 的图元数 |
| `exit` | `int` | 离开此 bin 的图元数 |

### `BVHSpatialStorage`
空间分割(Spatial Split)的线程本地存储，预分配内存以避免分割过程中的频繁内存操作。

| 成员 | 类型 | 说明 |
|------|------|------|
| `right_bounds` | `vector<BoundBox>` | 从右向左扫描时累积的包围盒 |
| `bins[3][NUM_SPATIAL_BINS]` | `BVHSpatialBin` | 3 个轴 x 32 个 bin 的直方图 |
| `new_references` | `vector<BVHReference>` | 空间分割产生的新引用的临时存储 |

## 核心函数

### `bvh_layout_name(BVHLayout layout)`
返回 BVH 布局的人类可读名称字符串。声明于此头文件中（实现在其他文件）。

### `BVHParams::best_bvh_layout(...)`
静态方法，根据请求的布局和设备支持的布局掩码，选择最佳匹配的 BVH 布局。优先使用请求的布局，若不支持则回退到支持的最宽布局。

## 依赖关系

- **内部头文件**:
  - `util/boundbox.h` - `BoundBox` 类
  - `util/vector.h` - `vector` 容器
  - `kernel/types.h` - `KernelBVHLayout` 类型定义
- **被引用**:
  - BVH 子系统: `bvh/bvh.h`、`bvh/bvh2.h`、`bvh/build.h`、`bvh/build.cpp`、`bvh/binning.h`、`bvh/multi.h`、`bvh/sort.cpp`、`bvh/split.h`、`bvh/unaligned.cpp`
  - 场景模块: `scene/scene.h`、`scene/geometry.h`、`scene/geometry_bvh.cpp`
  - 设备模块: `device/device.h`
  - 工具模块: `util/debug.h`

## 实现细节 / 关键算法

### 表面积启发式(SAH)开销模型
SAH 开销函数为 `C = N_node * C_node + N_prim * C_prim`，其中：
- `C_node` 是遍历一个节点的开销（默认 1.0）
- `C_prim` 是与一个图元进行相交测试的开销（默认 1.0）
- 该开销用于在构建过程中选择最优的分割平面

### BoundBox 打包技巧
`BVHReference` 和 `BVHRange` 将整数数据通过 `__int_as_float` / `__float_as_int` 编码到 `BoundBox` 的第四分量（`w`）中。这种技巧利用了 SIMD 对齐的 float4 结构，避免额外的内存开销，同时保持缓存行对齐。

### 空间分割存储预分配
`BVHSpatialStorage` 的设计目的是为每个线程提供独立的预分配存储空间。通过在构建初期预分配 bin 数组和临时引用容器，避免空间分割过程中大量的动态内存分配和 `memmove` 操作。

## 关联文件

- `bvh/build.h` / `bvh/build.cpp` - BVH 构建器，主要使用 `BVHParams` 和 `BVHSpatialStorage`
- `bvh/split.h` / `bvh/split.cpp` - 分割策略实现，使用 `BVHSpatialBin` 和 `BVHRange`
- `bvh/binning.h` - 分箱策略，使用 `BVHReference` 和 `BVHRange`
- `bvh/sort.h` - 引用排序，操作 `BVHReference` 数组
- `kernel/types.h` - 内核侧 BVH 布局类型定义
