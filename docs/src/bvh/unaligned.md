# unaligned.h / unaligned.cpp - 非对齐包围盒计算工具

## 概述

本模块实现了 BVH 构建中非对齐（Oriented/Unaligned）包围盒的计算功能。传统的轴对齐包围盒（AABB）对于曲线等细长的图元来说可能非常松散，导致大量无效的光线-图元相交测试。`BVHUnaligned` 类通过根据曲线段的方向构造定向包围盒（OBB），显著提升了曲线图元的包围盒紧密度，从而改善光线遍历效率。

## 类与结构体

### `BVHUnaligned`
非对齐包围盒计算的辅助类，提供对齐空间的计算和包围盒变换功能。

**构造函数:**

| 参数 | 类型 | 说明 |
|------|------|------|
| `objects` | `const vector<Object *> &` | BVH 所包含的对象列表 |

**公有方法:**

| 方法 | 说明 |
|------|------|
| `compute_aligned_space(range/binning, references)` | 为给定范围计算定向节点的对齐空间变换 |
| `compute_aligned_space(ref, aligned_space)` | 为单个引用计算对齐空间，成功返回 `true` |
| `compute_aligned_prim_boundbox(prim, aligned_space)` | 在给定对齐空间中计算图元的包围盒 |
| `compute_aligned_boundbox(range/binning, references, aligned_space, cent_bounds)` | 在给定对齐空间中计算范围内所有图元的包围盒 |
| `compute_node_transform(bounds, aligned_space)` (静态) | 计算节点打包用的仿射变换（将包围盒归一化到 0..1 范围） |

**保护成员:**

| 成员 | 类型 | 说明 |
|------|------|------|
| `objects_` | `const vector<Object *> &` | 对象列表引用 |

## 核心函数

### `BVHUnaligned::compute_aligned_space(const BVHReference &ref, Transform *aligned_space)`
为单个图元引用计算对齐空间变换矩阵。

**算法:**
1. 判断图元类型：仅对非运动模糊的曲线（`PRIMITIVE_CURVE` 且非 `PRIMITIVE_MOTION`）进行处理
2. 获取曲线段的两个控制点 `v1` 和 `v2`
3. 计算曲线段方向向量 `axis = normalize(v2 - v1)` 及其长度
4. 若长度 > 1e-6（非退化曲线段），使用 `make_transform_frame(axis)` 构造以该方向为基准的坐标系
5. 否则返回单位变换并返回 `false`

### `BVHUnaligned::compute_aligned_space(range, references)` (两个重载)
为一个范围（`BVHRange` 或 `BVHObjectBinning`）计算对齐空间：
- 遍历范围内的所有引用
- 使用第一个能成功计算方向的图元来定义整个范围的对齐空间
- 若没有合适的图元，返回单位变换

### `BVHUnaligned::compute_aligned_prim_boundbox(prim, aligned_space)`
在给定对齐空间中计算单个图元的包围盒。

**逻辑:**
- 对于非运动模糊曲线：使用 `curve.bounds_grow()` 在对齐空间中精确计算曲线段包围盒（考虑曲线半径）
- 对于其他图元类型：使用 `prim.bounds().transformed(&aligned_space)` 将轴对齐包围盒变换到对齐空间

### `BVHUnaligned::compute_aligned_boundbox(range, references, aligned_space, cent_bounds)`
在对齐空间中计算范围内所有图元的联合包围盒。遍历所有引用，对每个引用调用 `compute_aligned_prim_boundbox()`，并累积包围盒。可选地同时计算质心包围盒 `cent_bounds`。

两个重载版本分别接受 `BVHObjectBinning` 和 `BVHRange`，逻辑完全相同。

### `BVHUnaligned::compute_node_transform(bounds, aligned_space)` (静态)
计算将对齐空间包围盒归一化到 [0, 1] 范围的仿射变换：
1. 将对齐空间的平移分量减去包围盒最小值
2. 乘以包围盒尺寸的倒数缩放矩阵
3. 结果变换将包围盒映射到单位立方体，用于节点数据的紧凑存储

## 依赖关系

- **内部头文件**（unaligned.h）:
  - `util/vector.h` - `vector` 容器
- **内部头文件**（unaligned.cpp）:
  - `bvh/unaligned.h` - 本文件头文件
  - `bvh/binning.h` - `BVHObjectBinning` 类
  - `bvh/params.h` - `BVHReference`、`BVHRange` 类
  - `scene/hair.h` - `Hair::Curve` 曲线数据
  - `scene/object.h` - 场景对象
  - `util/boundbox.h` - `BoundBox` 类
  - `util/transform.h` - `Transform`、`transform_identity`、`make_transform_frame` 等
- **被引用**: `bvh/sort.cpp`、`bvh/build.h`、`bvh/bvh2.cpp`、`bvh/binning.h`

## 实现细节 / 关键算法

### 曲线段定向包围盒
对于曲线这类细长图元，轴对齐包围盒（AABB）的体积可能远大于实际图元。通过以曲线段方向为基准构造定向坐标系，然后在该坐标系中计算包围盒，可以得到更加紧密的包围体：
- 取曲线段两端控制点 `v1` 和 `v2`
- 以 `normalize(v2 - v1)` 为一个坐标轴
- 使用 `make_transform_frame()` 构造完整的正交基

### 运动模糊曲线的限制
代码注释明确指出"No motion blur curves here, we can't fit them to aligned boxes well"。运动模糊曲线因其在时间维度上的变形，难以用单一方向定义有效的定向包围盒，因此对运动模糊曲线使用标准的轴对齐包围盒。

### 范围内对齐空间的选择策略
对于包含多个图元的范围，使用第一个有效曲线段的方向定义整个范围的对齐空间。这是一种简化的启发式方法——理想情况下应基于所有图元的主方向计算（如 PCA），但使用第一个有效图元在实践中已足够有效且开销更低。

### 节点变换归一化
`compute_node_transform()` 将包围盒坐标归一化到 [0, 1]^3，用于 BVH 节点的紧凑存储。使用 `max(1e-18f, dim)` 避免除以零，处理退化（面积为零）的包围盒情况。

## 关联文件

- `bvh/build.h` / `bvh/build.cpp` - BVH 构建器，根据 `unaligned_split_threshold` 决定是否使用非对齐节点
- `bvh/split.h` / `bvh/split.cpp` - 分割策略，使用非对齐包围盒进行分割平面评估
- `bvh/sort.h` / `bvh/sort.cpp` - 排序时支持在非对齐空间中比较
- `bvh/binning.h` - 分箱时使用非对齐包围盒
- `bvh/bvh2.cpp` - BVH2 构建中使用非对齐节点
- `scene/hair.h` - 曲线数据访问
- `util/transform.h` - 变换矩阵工具函数
