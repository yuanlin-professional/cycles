# boundbox.h - 轴对齐包围盒(AABB)

## 概述

`boundbox.h` 定义了 Cycles 渲染器中的轴对齐包围盒(AABB)类 `BoundBox` 和 2D 包围盒类 `BoundBox2D`。BoundBox 是层次包围体(BVH)加速结构的基本构建块，在场景构建、光线追踪、光源裁剪等环节中被广泛使用。它由最小角点 `min` 和最大角点 `max` 两个 float3 定义。

## 核心函数

### BoundBox（3D 轴对齐包围盒）

#### 构造
| 函数 | 说明 |
|------|------|
| `BoundBox()` | 默认构造（未初始化） |
| `BoundBox(float3 pt)` | 单点构造（min=max=pt） |
| `BoundBox(float3 min, float3 max)` | 指定最小/最大角点 |
| `BoundBox(empty)` | 空包围盒（min=FLT_MAX, max=-FLT_MAX） |

#### 扩展与缩减
| 函数 | 说明 |
|------|------|
| `grow(float3 pt)` | 扩展以包含点 pt |
| `grow(float3 pt, float border)` | 扩展以包含点 pt 加边距 |
| `grow(BoundBox bbox)` | 扩展以包含另一个包围盒 |
| `grow_safe(...)` | 安全版本，忽略 NaN/Inf 输入 |
| `intersect(BoundBox bbox)` | 收缩为与另一包围盒的交集 |

#### 属性查询
| 函数 | 说明 |
|------|------|
| `area()` / `half_area()` | 表面积 / 半表面积（BVH 的 SAH 代价评估常用） |
| `safe_area()` | 安全表面积（无效包围盒返回 0） |
| `center()` / `center2()` | 中心点 / 中心点的 2 倍（避免除法） |
| `size()` | 三轴尺寸向量 |
| `valid()` | 有效性检查（min<=max 且有限） |
| `intersects(BoundBox other)` | 是否与另一包围盒相交 |
| `transformed(Transform* tfm)` | 经变换后的包围盒（对 8 个角点变换后重新计算） |

#### 自由函数
| 函数 | 说明 |
|------|------|
| `merge(BoundBox, float3)` | 合并包围盒与点 |
| `merge(BoundBox, BoundBox)` | 合并两个包围盒 |
| `merge(a, b, c, d)` | 合并四个包围盒 |
| `intersect(BoundBox, BoundBox)` | 求两个/三个包围盒的交集 |

### BoundBox2D（2D 包围盒）

| 函数 | 说明 |
|------|------|
| `width()` / `height()` | 宽度 / 高度 |
| `operator*(float)` | 标量缩放 |
| `subset(BoundBox2D)` | 在当前范围内取子区域 |
| `make_relative_to(BoundBox2D)` | 转换为相对坐标 |
| `clamp(mn, mx)` | 截断到 [mn, mx] |
| `offset(float2)` | 偏移 |

## 依赖关系

- **内部头文件**: `<cfloat>`, `<cmath>`, `util/math.h`, `util/transform.h`, `util/types.h`
- **被引用**: `bvh/node.h`, `bvh/octree.h`, `bvh/params.h`, `bvh/binning.cpp`, `bvh/unaligned.cpp`, `scene/geometry.h`, `scene/mesh.h`, `scene/object.h`, `scene/camera.h`, `scene/light_tree.h`, `subd/patch.h`

## 实现细节 / 关键算法

1. **NaN 安全的 grow**: `grow(pt)` 中 `min = ccl::min(pt, min)` 的参数顺序经过刻意安排——如果 `pt` 是 NaN，IEEE 754 规定 `min(NaN, x) = x`，因此 NaN 不会影响包围盒。

2. **`half_area`**: 计算公式为 `d.x*d.z + d.y*d.z + d.x*d.y`，即三对面各一个面的面积之和。这是 BVH 中 SAH (Surface Area Heuristic) 算法的核心度量，`half_area` 比 `area` 少一次乘法。

3. **`transformed`**: 对包围盒的 8 个角点分别应用变换矩阵，然后重新计算包含所有变换后角点的新包围盒。空包围盒直接返回空（避免将无效边界变换为有效）。

4. **`intersects`**: 使用中心点距离和半尺寸之和进行 AABB-AABB 碰撞检测，等价于但在某些场景下可能比直接比较 min/max 更快。

5. **`BoundBox2D`**: 与 3D 版本不同，使用 left/right/bottom/top 四个标量表示，用于相机裁剪窗口、渲染区域等 2D 场景。

## 关联文件

- `util/transform.h` — `Transform` 类型和 `transform_point` 函数
- `bvh/node.h` — BVH 节点存储 BoundBox 用于加速遍历
- `scene/geometry.h` — 几何体使用 BoundBox 进行空间划分
