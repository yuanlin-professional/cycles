# math_intersect.h - 光线与几何体求交

## 概述

`math_intersect.h` 提供光线与各种几何体的求交算法，是 Cycles 渲染器光线追踪核心的基础数学工具。支持的几何体包括球体、圆盘、三角形、四边形、平面、轴对齐包围盒(AABB)、无限圆柱体和圆锥体。这些函数主要在 kernel 的几何求交和光源采样模块中使用。

## 核心函数

### 球体与圆盘
| 函数 | 说明 |
|------|------|
| `ray_sphere_intersect` | 光线-球体求交，基于余弦定律求 t 值 |
| `ray_aligned_disk_intersect` | 光线-朝向光线原点的圆盘求交 |
| `ray_disk_intersect` | 光线-任意朝向圆盘求交 |

### 三角形
| 函数 | 说明 |
|------|------|
| `ray_triangle_intersect` | 光线-三角形求交（Plucker 坐标法，兼容 Embree） |
| `ray_triangle_intersect_self` | 自相交检测（用于避免光线原点三角形的误命中） |
| `ray_triangle_reciprocal` | 自定义倒数，匹配 Embree 位精度 |
| `ray_triangle_dot` / `ray_triangle_cross` | 自定义点积/叉积，SSE 下匹配 Embree 精度 |

### 四边形
| 函数 | 说明 |
|------|------|
| `ray_quad_intersect` | 光线-四边形求交，支持椭圆裁剪模式 |

### 体积几何体
| 函数 | 说明 |
|------|------|
| `ray_plane_intersect` | 光线-平面求交，返回法线正侧的 t 范围 |
| `ray_aabb_intersect` | 光线-轴对齐包围盒(AABB)求交，返回 t 区间 |
| `ray_infinite_cylinder_intersect` | 光线-无限椭圆柱体求交 |
| `ray_cone_intersect` | 光线-单侧圆锥体求交 |

## 依赖关系

- **内部头文件**: `util/math_float2.h`, `util/math_float3.h`, `util/math_float4.h`
- **被引用**: `kernel/geom/triangle_intersect.h`, `kernel/geom/motion_triangle_intersect.h`, `kernel/light/area.h`, `kernel/light/point.h`, `kernel/light/spot.h`, `kernel/light/triangle.h`, `kernel/light/background.h`, `kernel/integrator/shade_surface.h`

## 实现细节 / 关键算法

1. **三角形求交 (Plucker 坐标)**: 采用与 Embree 完全一致的 Plucker 坐标方法。将三角形顶点转换到光线原点的相对坐标系，通过边的 Plucker 坐标计算重心坐标 U/V/W。使用自定义的 `ray_triangle_dot`/`ray_triangle_cross` 确保与 Embree 位精度一致，这对混合使用 Embree 硬件加速和软件回退时的正确性至关重要。

2. **自相交检测**: `ray_triangle_intersect_self` 使用扩展的 epsilon 阈值（`minUVW >= eps` 而非 `>= -eps`），确保从三角形表面出发的光线不会误命中邻近三角形。

3. **AABB 求交**: 使用经典的 slab 方法——计算光线与三组平行平面的 t 区间，取 tmin 的最大值和 tmax 的最小值。利用 float4 的 `reduce_max`/`reduce_min` 一次性处理三个轴加已有的 t 范围。

4. **圆柱体求交**: 将 3D 问题投影到 2D 平面（xy 平面），求解标准二次方程。为避免光线远离圆柱时的精度问题，先将光线原点平移到最近点附近再求解。

5. **圆锥体求交**: 基于 Geometric Tools 文档的方法，求解关于圆锥轴方向分量和半角余弦的二次方程。近似退化为平面（半角接近 90 度）时切换到平面求交。需要额外检查交点是否在圆锥的正确半球内。

## 关联文件

- `util/math_base.h` — `Interval<T>` 结构体、`solve_quadratic` 求根函数
- `kernel/geom/triangle_intersect.h` — 三角形求交的上层调用者
- `kernel/light/area.h` — 面光源使用四边形求交
