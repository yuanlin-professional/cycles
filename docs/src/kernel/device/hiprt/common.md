# common.h - HIPRT 光线追踪载荷结构与自定义求交/过滤函数

## 概述

本文件是 HIPRT 后端的核心公共头文件，定义了光线遍历所需的载荷（payload）数据结构、光线转换宏、遍历栈初始化宏，以及全部的自定义图元求交函数和求交过滤函数。这些函数通过 HIPRT 的函数表机制（`intersectFunc` / `filterFunc`）被硬件遍历器回调，实现了对曲线、运动三角形、点云等非硬件原生图元的自定义求交逻辑。

## 核心函数/宏定义

### 载荷结构体

- **`RayPayload`** - 基础光线载荷，携带 `KernelGlobals`、自相交排除信息 `RaySelfPrimitives`、可见性标志、图元类型和光线时间。
- **`ShadowPayload`** - 继承自 `RayPayload`，增加积分器阴影状态、最大/当前透明命中计数、已记录命中数指针和透射率指针，用于阴影全记录求交。
- **`LocalPayload`** - 局部求交载荷，携带局部对象 ID、最大命中数、LCG 随机状态和 `LocalIntersection` 结果指针，用于子表面散射求交。

### 宏定义

- **`SET_HIPRT_RAY(RAY_RT, RAY)`** - 将 Cycles `Ray` 的方向、原点、tmax、tmin 复制到 `hiprtRay`。
- **`GET_TRAVERSAL_STACK()`** - 初始化全局栈 `Stack` 和实例栈 `Instance_Stack`，从 `KernelGlobals` 获取共享内存和全局栈缓冲区。
- **`GET_TRAVERSAL_ANY_HIT(FUNCTION_TABLE, RAY_TYPE, RAY_TIME)`** - 创建场景级 any-hit 遍历器 `hiprtSceneTraversalAnyHitCustomStack`。
- **`GET_TRAVERSAL_CLOSEST_HIT(FUNCTION_TABLE, RAY_TYPE, RAY_TIME)`** - 创建场景级 closest-hit 遍历器 `hiprtSceneTraversalClosestCustomStack`。

### 辅助函数

- **`set_intersect_point(kg, hit, isect)`** - 将 `hiprtHit` 结果转换为 Cycles `Intersection` 结构，进行实例 ID 到对象 ID 的映射和图元偏移计算。

### 自定义求交函数

- **`curve_custom_intersect()`** - 曲线图元自定义求交。从 `custom_prim_info` 获取曲线索引和类型，调用 `curve_intersect()` 计算精确求交，支持阴影链接和运动模糊时间过滤。
- **`motion_triangle_custom_intersect()`** - 运动三角形自定义求交，调用 `motion_triangle_intersect()`。
- **`motion_triangle_custom_local_intersect()`** - 运动三角形局部求交，用于子表面散射，受 `__OBJECT_MOTION__` 保护。
- **`motion_triangle_custom_volume_intersect()`** - 运动三角形体积求交，额外检查 `SD_OBJECT_HAS_VOLUME` 标志。
- **`point_custom_intersect()`** - 点云图元自定义求交，调用 `point_intersect()`，受 `__POINTCLOUD__` 保护。

### 求交过滤函数

- **`closest_intersection_filter()`** - 最近求交过滤器，跳过自相交和阴影链接排除的图元。
- **`shadow_intersection_filter()`** - 阴影求交过滤器，处理透明阴影记录、可见性检查和积分器状态数组写入。
- **`shadow_intersection_filter_curves()`** - 曲线阴影过滤器，额外处理曲线端点过滤（u==0 或 u==1）和曲线阴影透明度衰减。
- **`local_intersection_filter()`** - 局部求交过滤器，进行水库采样记录多个命中，计算几何法线。
- **`volume_intersection_filter()`** - 体积过滤器，仅接受具有体积属性的对象。

### HIPRT 回调入口

- **`intersectFunc(geom_type, ray_type, ...)`** - HIPRT 自定义求交回调分发器，根据 `geom_type * ray_type` 索引分发到对应的自定义求交函数。
- **`filterFunc(geom_type, ray_type, ...)`** - HIPRT 过滤回调分发器，根据索引分发到对应的过滤函数。

## 依赖关系

- **内部头文件**: 无显式 `#include`（本文件通过 `__HIPRT__` 宏保护，依赖由编译环境提供的 HIPRT 和 Cycles 内核头文件）

- **被引用**:
  - `kernel/device/hiprt/bvh.h` - BVH 求交实现直接包含本文件

## 实现细节 / 关键算法

1. **函数表索引机制**: HIPRT 使用二维索引 `index = numGeomTypes * ray_type + geom_type` 将不同的图元类型和光线类型映射到正确的自定义求交/过滤函数。索引值由 `globals.h` 中的枚举定义。

2. **ShadowPayload 继承设计**: `ShadowPayload` 继承自 `RayPayload`，利用了 `reinterpret_cast` 从 `void*` 到 `RayPayload*` 的兼容性，使得阴影和普通光线可以共享同一套自定义求交函数。

3. **透明阴影记录算法**: `shadow_intersection_filter` 在命中数超过 `INTEGRATOR_SHADOW_ISECT_SIZE` 时，采用替换最远命中的策略进行有限缓冲区管理。

4. **曲线端点过滤**: `shadow_intersection_filter_curves` 中，当 `u == 0.0f` 或 `u == 1.0f` 时忽略命中，避免曲线段端点处的重复计算。

5. **曲线阴影透明度截断**: 当曲线透射率低于 `CURVE_SHADOW_TRANSPARENCY_CUTOFF` 时终止遍历，认为光线已被完全遮挡。

## 关联文件

- `kernel/device/hiprt/bvh.h` - 场景求交函数，调用本文件中的宏和结构体
- `kernel/device/hiprt/globals.h` - 函数表索引枚举和内核参数结构
- `kernel/bvh/util.h` - BVH 工具函数（`bvh_instance_push` 等）
- `kernel/geom/curve_intersect.h` - 曲线求交核心算法
- `kernel/geom/motion_triangle_intersect.h` - 运动三角形求交核心算法
- `kernel/geom/point_intersect.h` - 点云求交核心算法
