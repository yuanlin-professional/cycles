# volume.h - 体积 BVH 遍历模板（单次命中）

## 概述

`volume.h` 是一个 BVH2 遍历模板文件，专门用于体积渲染中的光线交叉检测。与常规遍历不同，体积遍历仅关注具有体积属性的物体（`SD_OBJECT_HAS_VOLUME`），跳过非体积几何体以提高效率。该遍历用于初始化或更新体积栈，确定光线当前所处的体积区域。文件仅处理三角形图元（含运动模糊），不涉及曲线和点云的交叉。在 `__VOLUME__` 启用且 `__VOLUME_RECORD_ALL__` 未启用时编译。

## 类与结构体

本文件不定义新结构体，使用：

- **`Ray`**: 光线参数。
- **`Intersection`**: 交叉结果。

## 枚举与常量

| 宏 / 常量 | 说明 |
|---|---|
| `SD_OBJECT_HAS_VOLUME` | 物体标志，指示该物体包含体积数据 |
| `NODE_INTERSECT` | 根据 `BVH_HAIR` 特性选择节点交叉函数 |
| `BVH_MOTION` | 控制是否编译运动模糊三角形交叉 |

## 核心函数

### BVH_FUNCTION_FULL_NAME(BVH)()
- **签名**: `bool BVH_FUNCTION_FULL_NAME(BVH)(KernelGlobals kg, const ccl_private Ray *ray, ccl_private Intersection *isect, const uint visibility)`
- **功能**: 遍历场景 BVH 查找光线与体积物体表面的最近交叉点。遍历逻辑与常规 `traversal.h` 类似，但有以下关键差异：
  1. **体积过滤**: 在图元交叉前检查 `SD_OBJECT_HAS_VOLUME` 标志，跳过非体积物体的图元
  2. **实例过滤**: 进入实例节点时也检查体积标志，非体积实例直接跳过（弹栈）
  3. **图元限制**: 仅处理 `PRIMITIVE_TRIANGLE` 和 `PRIMITIVE_MOTION_TRIANGLE`
  4. **无阴影提前终止**: 不支持不透明阴影光线的提前返回

### BVH_FUNCTION_NAME()
- **签名**: `bool BVH_FUNCTION_NAME(KernelGlobals kg, const ccl_private Ray *ray, ccl_private Intersection *isect, const uint visibility)`
- **功能**: 内联包装函数。生成的具体名称为 `bvh_intersect_volume` 和 `bvh_intersect_volume_motion`。

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
- `src/kernel/bvh/bvh.h` - 在 `__VOLUME__` 且非 `__VOLUME_RECORD_ALL__` 条件下通过 `#include "kernel/bvh/volume.h"` 两次包含（普通版和运动模糊版）

## 实现细节 / 关键算法

### 体积过滤机制

体积遍历需要找到光线与体积物体边界表面的交叉，以确定光线何时进入或离开体积区域：

1. **图元级过滤**: 对每个叶节点中的三角形，先获取所属物体的标志，若无 `SD_OBJECT_HAS_VOLUME` 则 `continue`
2. **实例级过滤**: 遇到实例节点时，检查实例物体是否有体积属性；无体积的实例直接弹栈跳过，避免进入无用的子 BVH

这种双层过滤确保遍历只花费时间在体积相关的几何体上。

### 与 volume_all.h 的关系

- `volume.h` 查找**最近**的一个体积交叉点（`isect->t` 持续缩减）
- `volume_all.h` 记录**所有**体积交叉点（在 `__VOLUME_RECORD_ALL__` 模式下编译）

两者通过编译条件互斥：
- `__VOLUME__` && `!__VOLUME_RECORD_ALL__` -> `volume.h`
- `__VOLUME__` && `__VOLUME_RECORD_ALL__` -> `volume_all.h`

### 自交叉避免

使用 `intersection_skip_self()`（而非 `intersection_skip_self_shadow()`），因为体积遍历不涉及阴影光线，不需要额外检查光源图元。

### 遍历结构

采用与 `traversal.h` 相同的基于栈的双层循环遍历结构，支持实例化物体的 push/pop。初始化 `isect` 的所有字段（`t = tmax`, `u = 0`, `v = 0`, `prim = PRIM_NONE`, `object = OBJECT_NONE`），并通过 `isect->prim != PRIM_NONE` 判断是否找到有效交叉。

## 关联文件

| 文件 | 关系 |
|---|---|
| `kernel/bvh/bvh.h` | 包含本文件生成体积遍历函数 |
| `kernel/bvh/volume_all.h` | 互斥的多次命中体积遍历模板 |
| `kernel/bvh/traversal.h` | 结构类似的常规遍历模板 |
| `kernel/bvh/types.h` | 宏和常量定义 |
| `kernel/integrator/intersect_volume_stack.h` | 调用 `scene_intersect_volume()` |
