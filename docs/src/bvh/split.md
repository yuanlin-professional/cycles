# split.h / split.cpp - 层次包围体(BVH)分割策略实现

## 概述

本模块实现了层次包围体(BVH)构建中的三种分割策略：对象分割（Object Split）、空间分割(Spatial Split)和混合分割（Mixed Split）。对象分割按表面积启发式(SAH)将图元分为左右两组；空间分割(Spatial Split)沿空间平面切割图元，允许图元被复制到两侧；混合分割综合两者，选择 SAH 开销最低的方案。这些策略是 BVH 构建算法的核心决策组件，直接影响最终加速结构的遍历性能。

## 类与结构体

### `BVHObjectSplit`
对象分割策略类，通过表面积启发式(SAH)在三个坐标轴上寻找最优分割平面。

**公有成员:**

| 成员 | 类型 | 说明 |
|------|------|------|
| `sah` | `float` | 最优分割的 SAH 开销 |
| `dim` | `int` | 最优分割轴（0=X, 1=Y, 2=Z） |
| `num_left` | `int` | 左子节点的图元数量 |
| `left_bounds` | `BoundBox` | 左子节点包围盒 |
| `right_bounds` | `BoundBox` | 右子节点包围盒 |

**保护成员:**

| 成员 | 类型 | 说明 |
|------|------|------|
| `storage_` | `BVHSpatialStorage *` | 线程本地存储 |
| `references_` | `vector<BVHReference> *` | 图元引用数组指针 |
| `unaligned_heuristic_` | `const BVHUnaligned *` | 非对齐启发式 |
| `aligned_space_` | `const Transform *` | 对齐空间变换 |

### `BVHSpatialSplit`
空间分割(Spatial Split)策略类，沿空间平面切割图元（可复制跨越分割面的图元）。

**公有成员:**

| 成员 | 类型 | 说明 |
|------|------|------|
| `sah` | `float` | 最优分割的 SAH 开销 |
| `dim` | `int` | 最优分割轴 |
| `pos` | `float` | 分割平面位置 |

**核心方法:**

| 方法 | 说明 |
|------|------|
| `split(builder, left, right, range)` | 执行空间分割(Spatial Split)，将范围分为左右两部分 |
| `split_reference(builder, left, right, ref, dim, pos)` | 分割单个图元引用 |
| `split_triangle_primitive(...)` | 分割三角形图元 |
| `split_curve_primitive(...)` | 分割曲线图元 |
| `split_point_primitive(...)` | 分割点云图元 |
| `split_triangle_reference(...)` | 基于引用分割三角形 |
| `split_curve_reference(...)` | 基于引用分割曲线 |
| `split_point_reference(...)` | 基于引用分割点云 |
| `split_object_reference(...)` | 基于引用分割对象（遍历其所有图元） |

### `BVHMixedSplit`
混合分割策略类（仅在头文件中内联实现），综合对象分割和空间分割(Spatial Split)。

**公有成员:**

| 成员 | 类型 | 说明 |
|------|------|------|
| `object` | `BVHObjectSplit` | 对象分割结果 |
| `spatial` | `BVHSpatialSplit` | 空间分割(Spatial Split)结果 |
| `leafSAH` | `float` | 创建叶节点的 SAH 开销 |
| `nodeSAH` | `float` | 创建内部节点的 SAH 开销 |
| `minSAH` | `float` | 三者中最小的 SAH 开销 |
| `no_split` | `bool` | 是否应创建叶节点而非分割 |
| `bounds` | `BoundBox` | 当前范围的包围盒 |

## 核心函数

### `BVHObjectSplit::BVHObjectSplit(...)` (构造函数)
对象分割的最优平面搜索算法：
1. 遍历三个坐标轴
2. 在每个轴上对图元引用排序
3. 从右向左扫描，累积右侧包围盒
4. 从左向右扫描，对每个分割位置计算 SAH 开销：`SAH = nodeSAH + left_area * C_prim(left_count) + right_area * C_prim(right_count)`
5. 选择所有轴上 SAH 最小的分割方案

### `BVHObjectSplit::split(...)`
执行对象分割：按最优分割轴重新排序引用，然后根据 `num_left` 将引用数组分为左右两个 `BVHRange`。对于非对齐空间需要重新计算实际包围盒。

### `BVHSpatialSplit::BVHSpatialSplit(...)` (构造函数)
空间分割(Spatial Split)的最优平面搜索算法（基于 bin 的方法）：
1. 将范围的包围盒沿每个轴均匀分为 `NUM_SPATIAL_BINS`（32）个 bin
2. 将每个图元引用投射到 bin 中，记录每个 bin 的包围盒、进入和离开计数
3. 跨越多个 bin 的图元被切割，每段分别贡献到对应 bin
4. 从右向左扫描累积包围盒，从左向右扫描选择 SAH 最小的分割位置

### `BVHSpatialSplit::split(...)`
执行空间分割(Spatial Split)的核心逻辑：
1. **分类阶段**: 遍历所有引用，将完全在分割平面左侧的归入左组，完全在右侧的归入右组
2. **处理跨越图元**: 对跨越分割平面的图元，比较三种方案的 SAH 开销：
   - 不分割归入左组（unsplit left）
   - 不分割归入右组（unsplit right）
   - 复制为两个引用（duplicate），左右各一份
3. 选择 SAH 开销最小的方案
4. 复制的引用先存入临时数组 `new_references`，最后一次性插入实际数组以减少 `memmove` 调用

### `BVHSpatialSplit::split_reference(...)`
分割单个图元引用：
1. 根据图元类型（三角形、曲线、点云、对象）调用对应的分割函数
2. 用分割平面位置截断左右包围盒的边界
3. 与原始包围盒求交，确保分割后的包围盒不超出原始范围

### `BVHSpatialSplit::split_triangle_primitive(...)`
三角形的精确空间分割(Spatial Split)：
- 遍历三角形的三条边
- 将顶点归入对应的左/右包围盒
- 当边与分割平面相交时，计算交点并添加到左右两个包围盒

### `BVHMixedSplit::BVHMixedSplit(...)` (构造函数)
混合分割决策逻辑：
1. 先执行对象分割
2. 检查是否应使用空间分割(Spatial Split)：必须启用空间分割(Spatial Split)、深度未超过 `MAX_SPATIAL_DEPTH`、且左右子节点的包围盒重叠面积 >= `spatial_min_overlap`
3. 比较叶节点 SAH、对象分割 SAH 和空间分割 SAH，选择最小者
4. 若叶节点 SAH 最小且范围在最大叶节点大小内，标记为不分割

### `BVHMixedSplit::split(...)`
执行最终分割：优先尝试空间分割(Spatial Split)，若空间分割(Spatial Split)结果为空则回退到对象分割。

## 依赖关系

- **内部头文件**（split.h）:
  - `bvh/build.h` - `BVHBuild` 构建器类
  - `bvh/params.h` - `BVHParams`、`BVHReference`、`BVHRange`、`BVHSpatialStorage`
- **内部头文件**（split.cpp）:
  - `bvh/split.h` - 本文件头文件
  - `bvh/build.h` - 构建器（访问 `objects` 列表和参数）
  - `bvh/sort.h` - `bvh_reference_sort()` 排序函数
  - `scene/hair.h` - 曲线几何体
  - `scene/mesh.h` - 网格几何体
  - `scene/object.h` - 场景对象
  - `scene/pointcloud.h` - 点云几何体
  - `util/algorithm.h` - `swap()` 等标准算法
- **被引用**: `bvh/build.cpp`

## 实现细节 / 关键算法

### 表面积启发式(SAH)扫描优化
对象分割使用经典的双向扫描技术：
1. 先从右向左扫描，在 `right_bounds` 数组中累积每个分割位置的右侧包围盒
2. 再从左向右扫描，逐步扩展左侧包围盒，同时查表获取右侧包围盒
3. 每次排序后调用 `storage_->right_bounds.resize()` 是因为排序可能触发任务池中另一个线程的 `BVHObjectSplit` 使用同一存储并调整大小

### 空间分割(Spatial Split)的 bin 方法
空间分割使用 32 个 bin 的直方图方法：
- 沿每个轴将空间均匀分为 32 个 bin
- 将图元引用投射到 bin 中，跨越多个 bin 的图元在 bin 边界处被分割
- 使用 enter/exit 计数器跟踪每个 bin 中的图元数量
- 通过双向扫描选择 SAH 最优的分割位置

### 复制(Duplicate) vs 不分割(Unsplit) 决策
当图元跨越分割平面时，对每个图元独立比较三种 SAH 开销：
- `unsplitLeftSAH`: 将整个图元放入左侧
- `unsplitRightSAH`: 将整个图元放入右侧
- `duplicateSAH`: 将图元分割为两部分，分别放入左右两侧

这确保了仅在复制确实降低了总 SAH 开销时才增加引用数量。

### 图元类型专用分割
- **三角形**: 精确计算边与分割平面的交点
- **曲线**: 使用控制点进行线性插值（注释指出当前忽略了曲线宽度，需修复）
- **点云**: 不进行真正的分割，假设点足够小可忽略
- **对象**: 遍历对象内所有图元进行逐一分割

## 关联文件

- `bvh/build.h` / `bvh/build.cpp` - BVH 构建器，使用 `BVHMixedSplit` 作为分割决策
- `bvh/params.h` - 参数定义和基础数据结构
- `bvh/sort.h` / `bvh/sort.cpp` - 图元引用排序
- `bvh/unaligned.h` - 非对齐包围盒计算
- `scene/mesh.h` - 三角形网格，用于三角形精确分割
- `scene/hair.h` - 曲线几何体，用于曲线分割
- `scene/pointcloud.h` - 点云几何体，用于点分割
