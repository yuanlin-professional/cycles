# bvh.h - HIPRT 光线追踪场景求交实现

## 概述

本文件实现了基于 AMD HIPRT 硬件加速的光线-场景求交接口。它为 Cycles 渲染器的路径追踪积分器提供了完整的场景求交函数集合，包括最近交点查询、阴影光线求交、局部子表面散射求交、阴影全记录求交以及体积求交。所有求交函数均通过 HIPRT 的遍历 API（`hiprtSceneTraversal`）驱动硬件层次包围体（BVH）加速结构进行光线遍历。

## 核心函数/宏定义

### 辅助函数

- **`scene_intersect_valid(const Ray *ray)`** - 校验光线有效性，确保光线原点和方向均为有限值且方向非零。

### 场景求交函数

- **`scene_intersect(KernelGlobals kg, const Ray *ray, uint visibility, Intersection *isect)`** - 最近交点求交。对不透明阴影光线使用 any-hit 遍历，对其他光线使用 closest-hit 遍历。将 Cycles 光线转换为 `hiprtRay`，通过 `RayPayload` 携带内核全局数据和可见性标志。

- **`scene_intersect_shadow(KernelGlobals kg, const Ray *ray, uint visibility)`** - 阴影光线求交的简化封装，内部调用 `scene_intersect`。

- **`scene_intersect_local<single_hit>()`** - 局部求交，用于子表面散射（SSS）。仅处理三角形和运动三角形图元。对非硬件加速的运动三角形使用 `hiprtGeomCustomTraversalAnyHit`，对普通三角形使用 `hiprtGeomTraversalAnyHit`。需要将光线变换到对象局部空间。受 `__BVH_LOCAL__` 宏保护。

- **`scene_intersect_shadow_all()`** - 阴影全记录求交，记录路径上所有透明交点。使用 `ShadowPayload` 携带积分器状态及透明度计数。受 `__SHADOW_RECORD_ALL__` 宏保护。

- **`scene_intersect_volume()`** - 体积求交，仅对具有体积属性的对象进行求交。使用专用函数表 `table_volume_intersect`。受 `__VOLUME__` 宏保护。

### 宏定义

- **`SET_HIPRT_RAY`** - 将 Cycles 内部 `Ray` 结构转换为 `hiprtRay`。
- **`GET_TRAVERSAL_STACK`** / **`GET_TRAVERSAL_ANY_HIT`** / **`GET_TRAVERSAL_CLOSEST_HIT`** - 遍历栈初始化与遍历器创建宏（定义于 `common.h`，在本文件中被使用）。

## 依赖关系

- **内部头文件**:
  - `kernel/device/hiprt/common.h` - 提供 payload 结构体、遍历宏定义、自定义求交与过滤函数

- **被引用**:
  - `kernel/bvh/bvh.h` - BVH 抽象层根据编译后端条件引入本文件

## 实现细节 / 关键算法

1. **遍历策略选择**: 对不透明阴影光线（`PATH_RAY_SHADOW_OPAQUE`）使用 any-hit 遍历以提前终止，对普通光线使用 closest-hit 遍历以获取最近交点。

2. **曲线图元特殊处理**: 当交点类型 `isect->type > 1` 时，表示命中的是曲线图元，需要从 `payload.prim_type` 获取打包的曲线类型，并从 `hit.primID` 获取图元索引。

3. **局部求交的对象空间变换**: `scene_intersect_local` 在遍历前将光线从世界空间变换到对象局部空间（通过 `bvh_instance_push` 或 `bvh_instance_motion_push`），因为局部 BVH 几何体存储在对象空间中。

4. **函数表路由**: 不同的求交类型使用不同的 HIPRT 函数表（`table_closest_intersect`、`table_shadow_intersect`、`table_local_intersect`、`table_volume_intersect`），通过 `ray_type` 参数区分自定义求交和过滤函数的调用。

## 关联文件

- `kernel/device/hiprt/common.h` - payload 结构体与自定义求交/过滤函数
- `kernel/device/hiprt/globals.h` - 全局数据结构与函数表索引枚举
- `kernel/bvh/bvh.h` - BVH 抽象层入口
- `kernel/bvh/types.h` - BVH 类型定义
