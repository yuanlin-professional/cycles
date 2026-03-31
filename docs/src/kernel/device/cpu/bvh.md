# bvh.h - CPU 端基于 Embree 的光线-场景求交实现

## 概述

本文件实现了 Cycles 渲染器在 CPU 设备上使用 Intel Embree 库进行光线与场景求交的完整逻辑。它定义了多种求交上下文结构体（用于首次命中、阴影、局部子表面散射和体积求交），以及一系列 Embree 过滤回调函数和高层求交入口函数。该文件同时兼容 CPU 和 oneAPI 两种后端，通过条件编译在两者间切换。

## 核心函数/宏定义

### 求交上下文结构体
- **`CCLFirstHitContext`**: 继承自 `RTCRayQueryContext`，用于标准光线求交（最近命中），包含 `KernelGlobals` 和当前光线指针，用于自相交检测。
- **`CCLShadowContext`**: 用于阴影光线求交，记录透明命中次数、透过率、最大距离等信息，支持透明阴影的多次命中记录。
- **`CCLLocalContext`**: 用于局部求交（如子表面散射 SSS），支持水库采样算法在有限命中槽位中随机选取命中点。
- **`CCLVolumeContext`**: 用于体积对象求交，记录所有与体积对象的交点。

### 工具函数
- **`kernel_embree_setup_ray()`**: 将 Cycles 内部的 `Ray` 转换为 Embree 的 `RTCRay` 格式。
- **`kernel_embree_setup_rayhit()`**: 设置 `RTCRayHit`，初始化几何 ID 为无效值。
- **`kernel_embree_get_hit_object()`**: 从 Embree 命中结果中提取 Cycles 的对象 ID。
- **`kernel_embree_is_self_intersection()`**: 检测是否为自相交，避免光线与自身几何体重复命中。
- **`kernel_embree_convert_hit()`**: 将 Embree 命中结果转换为 Cycles 内部的 `Intersection` 结构，处理三角形和曲线两种图元类型。
- **`kernel_embree_convert_sss_hit()`**: 专用于 SSS 的命中转换。

### 过滤回调函数
- **`kernel_embree_filter_intersection_func_impl()`**: 标准求交过滤器，过滤自相交和阴影链接。
- **`kernel_embree_filter_occluded_shadow_all_func_impl()`**: 阴影光线过滤器，处理透明阴影、曲线阴影透明度，并使用最近 N 次命中替换策略记录交点。
- **`kernel_embree_filter_occluded_local_func_impl()`**: 局部求交过滤器，用于 SSS，实现水库采样（reservoir sampling）算法。
- **`kernel_embree_filter_occluded_volume_all_func_impl()`**: 体积求交过滤器，仅记录具有体积属性的对象交点。

### 高层求交入口
- **`kernel_embree_intersect()`**: 执行标准光线-场景最近命中求交。
- **`kernel_embree_intersect_local()`**: 执行局部对象求交（用于 SSS），支持对象自有层次包围体(BVH)的实例变换。
- **`kernel_embree_intersect_shadow_all()`**: 执行阴影光线求交，记录所有透明交点。
- **`kernel_embree_intersect_volume()`**: 执行体积求交，收集所有体积边界交点。

### 兼容性宏
- **`RTCTraversable` 兼容宏**: 对 Embree 4.4 之前版本定义 `RTCTraversable` 等别名，统一新旧 API。
- **`CYCLES_EMBREE_USED_FEATURES`**: 定义 Embree 使用的特性标志（三角形、实例、过滤函数、点、运动模糊、曲线等）。
- **`EMBREE_IS_HAIR(x)`**: 通过几何 ID 最低位判断图元是否为毛发。
- **`numhit_t`**: CPU 上为 `uint32_t`，oneAPI 上为 `uint16_t`。

## 依赖关系

- **内部头文件**:
  - `embree4/rtcore_geometry.h`、`embree4/rtcore_ray.h`、`embree4/rtcore_scene.h` (Embree 库)
  - `kernel/device/cpu/compat.h` (CPU 兼容层)
  - `kernel/device/cpu/globals.h` (CPU 全局数据)
  - `kernel/bvh/types.h`、`kernel/bvh/util.h` (层次包围体(BVH)类型与工具)
  - `kernel/geom/object.h` (几何对象操作)
  - `kernel/integrator/state.h`、`kernel/integrator/state_util.h` (积分器状态)
  - `kernel/sample/lcg.h` (线性同余随机数，用于水库采样)
- **被引用**: `src/kernel/bvh/bvh.h`

## 实现细节 / 关键算法

### 透明阴影命中记录策略
在 `kernel_embree_filter_occluded_shadow_all_func_impl` 中实现了一种最近 N 次命中的记录策略：当命中数超过 `INTEGRATOR_SHADOW_ISECT_SIZE` 时，用距离更近的命中替换记录中距离最远的命中，同时维护 `max_t` 缓存以快速跳过不可能更近的命中。

### 水库采样 (Reservoir Sampling)
在 `kernel_embree_filter_occluded_local_func_impl` 中用于 SSS 局部求交：当命中数超过最大槽位时，以概率 `max_hits / num_hits` 替换已有记录，确保所有命中点被均匀采样。

### Embree Traversable 兼容
为兼容 Embree 4.4 前后版本差异，对旧版本将 `RTCScene` 映射为 `RTCTraversable`，将 `rtcIntersect1` / `rtcOccluded1` 映射为对应的 Traversable 调用。

## 关联文件

- `src/kernel/bvh/bvh.h` — 层次包围体(BVH)遍历的顶层头文件，根据是否使用 Embree 选择不同求交实现
- `src/kernel/device/cpu/compat.h` — CPU 兼容性定义
- `src/kernel/device/cpu/globals.h` — CPU 设备全局数据结构
- `src/kernel/device/oneapi/compat.h` — oneAPI 兼容层（条件编译分支）
