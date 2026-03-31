# volume_all.h - 体积 BVH 遍历模板（多次命中记录）

## 概述

`volume_all.h` 是一个 BVH2 遍历模板文件，实现了体积渲染中记录所有交叉点的遍历逻辑。与 `volume.h`（只找最近交叉点）不同，本文件的遍历会收集光线路径上的多个体积边界交叉，将结果存入交叉数组中，用于正确构建和更新体积栈。该文件仅在 `__VOLUME__` 和 `__VOLUME_RECORD_ALL__` 同时启用时编译，与 `volume.h` 互斥。

## 类与结构体

本文件不定义新结构体，使用：

- **`Ray`**: 光线参数。
- **`Intersection`**: 交叉结果，本文件使用 `Intersection` 数组记录多次命中。

## 枚举与常量

| 宏 / 常量 | 说明 |
|---|---|
| `SD_OBJECT_HAS_VOLUME` | 物体标志，指示物体包含体积数据 |
| `NODE_INTERSECT` | 根据 `BVH_HAIR` 特性选择节点交叉函数 |
| `BVH_MOTION` | 控制运动模糊三角形交叉 |

## 核心函数

### BVH_FUNCTION_FULL_NAME(BVH)()
- **签名**: `uint BVH_FUNCTION_FULL_NAME(BVH)(KernelGlobals kg, const ccl_private Ray *ray, Intersection *isect_array, const uint max_hits, const uint visibility)`
- **功能**: 遍历场景 BVH，收集光线与体积物体表面的所有交叉点，最多记录 `max_hits` 个。与 `volume.h` 的关键差异：
  1. **返回值**: 返回 `uint` 类型的命中数量（而非 `bool`）
  2. **数组记录**: 使用 `isect_array` 指针递增方式记录每次命中，每命中一次 `isect_array++`
  3. **固定搜索范围**: 使用固定的 `isect_t = ray->tmax` 作为遍历范围，不随命中缩减
  4. **提前容量终止**: 当 `num_hits == max_hits` 时立即返回，不再继续遍历

  体积过滤逻辑与 `volume.h` 相同：仅处理 `SD_OBJECT_HAS_VOLUME` 标志的物体图元。

### BVH_FUNCTION_NAME()
- **签名**: `uint BVH_FUNCTION_NAME(KernelGlobals kg, const ccl_private Ray *ray, Intersection *isect_array, const uint max_hits, const uint visibility)`
- **功能**: 内联包装函数。生成的具体名称为 `bvh_intersect_volume_all` 和 `bvh_intersect_volume_all_motion`。

## 依赖关系

### 内部头文件
作为模板文件被 `#include`，依赖由 `bvh.h` 提前引入的：
- `kernel/bvh/types.h` - 宏定义
- `kernel/bvh/nodes.h` - 节点交叉函数
- `kernel/bvh/util.h` - `intersection_skip_self()`
- `kernel/geom/triangle_intersect.h` - `triangle_intersect()`
- `kernel/geom/motion_triangle_intersect.h` - `motion_triangle_intersect()`
- `kernel/geom/object.h` - 实例变换函数

### 被引用
- `src/kernel/bvh/bvh.h` - 在 `__VOLUME__` 且 `__VOLUME_RECORD_ALL__` 条件下通过 `#include "kernel/bvh/volume_all.h"` 两次包含（普通版和运动模糊版）

## 实现细节 / 关键算法

### 多次命中记录机制

交叉结果的记录采用数组指针递增方式：

```
isect_array[0] -> 第 1 次命中
isect_array[1] -> 第 2 次命中
...
isect_array[num_hits-1] -> 第 N 次命中
```

每次三角形交叉测试成功后：
1. 当前 `isect_array` 指针处已由 `triangle_intersect()` 写入交叉结果
2. 指针递增 `isect_array++`
3. 命中计数 `num_hits++`
4. 新位置的 `t` 设置为 `isect_t`（保持全范围搜索）
5. 达到 `max_hits` 时立即返回

### 与 volume.h 的对比

| 特性 | volume.h | volume_all.h |
|---|---|---|
| 返回类型 | `bool` | `uint`（命中数） |
| 命中记录 | 单个 `Intersection` | `Intersection` 数组 |
| 搜索范围 | 随命中缩减 `isect->t` | 固定为 `ray->tmax` |
| 编译条件 | `!__VOLUME_RECORD_ALL__` | `__VOLUME_RECORD_ALL__` |
| 用途 | 查找最近体积边界 | 收集所有体积边界交叉 |

### 体积栈构建

体积渲染需要知道光线穿过了哪些体积物体的边界。通过收集所有体积交叉点，积分器可以：
1. 确定光线在每个区间段内处于哪些体积内部
2. 正确叠加嵌套体积的属性
3. 处理体积重叠区域

### 实例过滤

与 `volume.h` 相同，对实例节点进行体积标志过滤：
- 有 `SD_OBJECT_HAS_VOLUME`: 进入实例 BVH 遍历
- 无体积标志: 直接弹栈跳过，设置 `object = OBJECT_NONE`

### 实例内交叉的 tmax 维护

进入实例 BVH 时，设置 `isect_array->t = isect_t`，确保实例内的三角形交叉测试使用正确的最大距离。

## 关联文件

| 文件 | 关系 |
|---|---|
| `kernel/bvh/bvh.h` | 包含本文件生成体积遍历函数 |
| `kernel/bvh/volume.h` | 互斥的单次命中体积遍历模板 |
| `kernel/bvh/traversal.h` | 结构类似的常规遍历模板 |
| `kernel/bvh/types.h` | 宏和常量定义 |
| `kernel/integrator/intersect_volume_stack.h` | 调用 `scene_intersect_volume()` |
