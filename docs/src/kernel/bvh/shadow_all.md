# shadow_all.h - 透明阴影全交叉记录 BVH 遍历模板

## 概述

`shadow_all.h` 是一个 BVH2 遍历模板文件，专门用于透明阴影光线的全路径交叉记录。与常规遍历只需找到最近交叉点不同，本文件的遍历会记录光线路径上所有的交叉点，以便后续计算透明材质的阴影衰减。该实现支持提前终止（不透明遮挡或超过最大透明弹射次数），以及曲线的烘焙阴影透明度快速路径。文件通过宏模板机制被 `bvh.h` 多次包含，生成针对不同图元组合的优化变体。

## 类与结构体

本文件不定义新的结构体，但使用：

- **`Intersection`**: 临时交叉结果，用于存储每次图元命中的信息。
- **`IntegratorShadowState`**: 积分器阴影状态，存储已记录交叉点的数组（`shadow_isect`）。
- **`Ray`**: 光线数据。

## 枚举与常量

| 宏 / 常量 | 说明 |
|---|---|
| `INTEGRATOR_SHADOW_ISECT_SIZE` | 阴影交叉记录数组的最大容量 |
| `CURVE_SHADOW_TRANSPARENCY_CUTOFF` (0.001f) | 曲线阴影透明度截止阈值 |
| `SD_HAS_TRANSPARENT_SHADOW` | 着色器标志，指示材质具有透明阴影 |
| `SD_HAS_ONLY_VOLUME` | 着色器标志，指示材质仅为体积边界 |
| `NODE_INTERSECT` | 根据 `BVH_HAIR` 特性选择节点交叉函数 |

## 核心函数

### BVH_FUNCTION_FULL_NAME(BVH)()
- **签名**: `bool BVH_FUNCTION_FULL_NAME(BVH)(KernelGlobals kg, const ccl_private Ray *ray, IntegratorShadowState state, const uint visibility, const uint max_transparent_hits, ccl_private uint *r_num_recorded_hits, ccl_private float *r_throughput)`
- **功能**: 遍历整个场景 BVH，记录阴影光线路径上的所有交叉点。对每个命中的图元：
  1. 检查着色器是否具有透明阴影（`SD_HAS_TRANSPARENT_SHADOW`），不透明材质立即返回 `true`（完全遮挡）
  2. 检查交叉是否已被记录（处理 BVH 空间分割导致的重复命中）
  3. 统计透明弹射次数，超过 `max_transparent_hits` 则视为完全遮挡
  4. 对曲线图元使用烘焙阴影透明度快速路径（直接更新 `throughput`，不记录交叉）
  5. 其他透明交叉写入 `IntegratorShadowState` 的 `shadow_isect` 数组
  6. 当记录数超过最大容量时，替换最远的交叉点以保留 N 个最近交叉

  参数说明：
  - `r_num_recorded_hits`: 输出已记录的命中总数（可能超过数组容量）
  - `r_throughput`: 输出曲线阴影衰减后的透射率

### BVH_FUNCTION_NAME()
- **签名**: `bool BVH_FUNCTION_NAME(KernelGlobals kg, const ccl_private Ray *ray, IntegratorShadowState state, const uint visibility, const uint max_transparent_hits, ccl_private uint *num_recorded_hits, ccl_private float *throughput)`
- **功能**: 内联包装函数。生成的具体名称包括：`bvh_intersect_shadow_all`、`bvh_intersect_shadow_all_hair`、`bvh_intersect_shadow_all_motion`、`bvh_intersect_shadow_all_hair_motion`。

## 依赖关系

### 内部头文件
作为模板文件被 `#include`，依赖由 `bvh.h` 提前引入的所有头文件，特别是：
- `kernel/bvh/types.h` - 宏定义
- `kernel/bvh/nodes.h` - 节点交叉函数
- `kernel/bvh/util.h` - `intersection_skip_self_shadow()`、`intersection_skip_shadow_link()`、`intersection_get_shader_flags()`、`intersection_curve_shadow_transparency()`、`intersection_skip_shadow_already_recoded()`
- `kernel/geom/triangle_intersect.h` - 三角形交叉
- `kernel/geom/curve_intersect.h` - 曲线交叉
- `kernel/geom/point_intersect.h` - 点交叉

### 被引用
- `src/kernel/bvh/bvh.h` - 通过 `#include "kernel/bvh/shadow_all.h"` 四次包含（基本版、毛发版、运动模糊版、毛发+运动模糊版）

## 实现细节 / 关键算法

### 全交叉记录策略

与最近交叉遍历的关键区别：
1. **不缩减搜索范围**: 常规遍历在找到更近交叉后会更新 `tmax`；本遍历使用固定的 `tmax`（光线最大距离），但维护一个 `tmax_hits`（已记录最远交叉距离）用于替换策略。
2. **多次命中记录**: 使用 `IntegratorShadowState` 中的 `shadow_isect` 数组存储交叉结果。
3. **有限容量管理**: 当记录数达到 `INTEGRATOR_SHADOW_ISECT_SIZE` 上限后，采用"替换最远"策略——找到数组中距离最大的条目并用新的更近交叉替换。

### 提前终止条件

遍历在以下情况下提前返回 `true`（完全遮挡）：
1. 命中不透明材质（无 `SD_HAS_TRANSPARENT_SHADOW` 标志）
2. 透明弹射次数超过 `max_transparent_hits`
3. 曲线阴影透射率低于 `CURVE_SHADOW_TRANSPARENCY_CUTOFF`（0.001）

### 曲线阴影快速路径

对于曲线图元，直接使用 `intersection_curve_shadow_transparency()` 获取烘焙的阴影透明度值并累乘到 `throughput` 中，而不记录到交叉数组。这避免了对曲线执行完整的着色器求值。

### 阴影链接（Shadow Linking）

当启用 `__SHADOW_LINKING__` 特性时，通过 `intersection_skip_shadow_link()` 检查物体是否在光源的阴影集合中，实现选择性阴影投射。

### 实例遍历

支持实例化物体的遍历，通过 `bvh_instance_push`/`bvh_instance_pop` 在进入/离开实例 BVH 时转换光线坐标。使用 `ENTRYPOINT_SENTINEL` 标记实例 BVH 的栈边界。

## 关联文件

| 文件 | 关系 |
|---|---|
| `kernel/bvh/bvh.h` | 包含本文件生成阴影遍历函数 |
| `kernel/bvh/traversal.h` | 结构类似的常规遍历模板 |
| `kernel/bvh/types.h` | 宏和常量定义 |
| `kernel/bvh/util.h` | 阴影相关工具函数 |
| `kernel/integrator/intersect_shadow.h` | 调用 `scene_intersect_shadow_all()` |
| `kernel/integrator/shade_shadow.h` | 处理记录的阴影交叉结果 |
