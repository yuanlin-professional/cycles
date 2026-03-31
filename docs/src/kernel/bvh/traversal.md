# traversal.h - 常规场景 BVH2 遍历模板

## 概述

`traversal.h` 是 Cycles 渲染器中 BVH2 光线遍历的核心模板文件，实现了在整个场景中查找光线最近交叉点的通用遍历逻辑。该文件通过预处理器宏模板机制支持多种图元特性（毛发曲线、点云、运动模糊）的编译期组合优化，被 `bvh.h` 多次 `#include` 以生成 4 个特化变体。其设计基于 "Understanding the Efficiency of Ray Traversal on GPUs" 论文的思想，经过扩展以支持 CPU 和多种 GPU 内核语言。

## 类与结构体

本文件不定义新结构体，使用：

- **`Ray`**: 光线参数。
- **`Intersection`**: 交叉结果（`t`、`u`、`v`、`prim`、`object`、`type`）。

## 枚举与常量

| 宏 / 常量 | 说明 |
|---|---|
| `BVH_HAIR` | 启用毛发曲线图元交叉支持 |
| `BVH_POINTCLOUD` | 启用点云图元交叉支持 |
| `BVH_MOTION` | 启用运动模糊图元交叉支持 |
| `NODE_INTERSECT` | 根据 `BVH_HAIR` 选择 `bvh_node_intersect`（支持 OBB）或 `bvh_aligned_node_intersect`（仅 AABB） |
| `PATH_RAY_SHADOW_OPAQUE` | 不透明阴影光线可见性标志，用于提前终止 |
| `OBJECT_NONE` / `PRIM_NONE` | 无效物体/图元索引哨兵值 |

## 核心函数

### BVH_FUNCTION_FULL_NAME(BVH)()
- **签名**: `ccl_device_noinline bool BVH_FUNCTION_FULL_NAME(BVH)(KernelGlobals kg, const ccl_private Ray *ray, ccl_private Intersection *isect, const uint visibility)`
- **功能**: 从 BVH 根节点开始遍历整个场景，找到光线的最近交叉点。支持的图元类型包括：
  - `PRIMITIVE_TRIANGLE` - 静态三角形
  - `PRIMITIVE_MOTION_TRIANGLE` - 运动模糊三角形（需 `BVH_MOTION`）
  - `PRIMITIVE_CURVE_*` - 多种曲线类型（需 `BVH_HAIR` + `__HAIR__`）
  - `PRIMITIVE_POINT` / `PRIMITIVE_MOTION_POINT` - 点云（需 `BVH_POINTCLOUD` + `__POINTCLOUD__`）

  遍历过程中持续更新 `isect->t` 为当前最近距离，用于剪枝加速。对于不透明阴影光线（`PATH_RAY_SHADOW_OPAQUE`），找到任意命中即可提前返回。

### BVH_FUNCTION_NAME()
- **签名**: `ccl_device_inline bool BVH_FUNCTION_NAME(KernelGlobals kg, const ccl_private Ray *ray, ccl_private Intersection *isect, const uint visibility)`
- **功能**: 内联包装函数。由 `bvh.h` 生成的具体名称包括：`bvh_intersect`、`bvh_intersect_hair`、`bvh_intersect_motion`、`bvh_intersect_hair_motion`。

## 依赖关系

### 内部头文件
作为模板文件被 `#include`，依赖由 `bvh.h` 提前引入的：
- `kernel/bvh/types.h` - 宏和常量
- `kernel/bvh/nodes.h` - 节点交叉测试
- `kernel/bvh/util.h` - 工具函数（`intersection_skip_self_shadow`、`intersection_skip_shadow_link`）
- `kernel/geom/triangle_intersect.h` - `triangle_intersect()`
- `kernel/geom/motion_triangle_intersect.h` - `motion_triangle_intersect()`
- `kernel/geom/curve_intersect.h` - `curve_intersect()`
- `kernel/geom/point_intersect.h` - `point_intersect()`
- `kernel/geom/object.h` - `bvh_instance_push()`、`bvh_instance_pop()`

### 被引用
- `src/kernel/bvh/bvh.h` - 通过 `#include "kernel/bvh/traversal.h"` 四次包含（四种特性组合）

## 实现细节 / 关键算法

### BVH2 遍历算法

采用基于栈的深度优先遍历：

```
初始化: stack = [ENTRYPOINT_SENTINEL], node = root
循环:
  1. 内部节点: 测试光线与两个子包围盒交叉
     - 两个都命中: 将较远子节点压栈，继续遍历较近子节点
     - 一个命中: 继续遍历命中的子节点
     - 都未命中: 从栈中弹出下一个节点
  2. 叶节点 (node_addr < 0): 遍历图元列表，执行精确交叉测试
     - 叶节点地址 >= 0: 图元列表
     - 叶节点地址 < 0: 实例节点，进入实例 BVH
  3. 遇到 ENTRYPOINT_SENTINEL: 实例 BVH 遍历结束，弹出到父级
```

### 实例化物体处理

遍历到实例节点时的处理流程：
1. **实例压入 (Instance Push)**: 获取实例物体 ID，将光线从世界空间变换到物体局部空间，压入 `ENTRYPOINT_SENTINEL` 作为实例 BVH 的栈边界
2. **实例内遍历**: 在物体局部 BVH 中正常遍历
3. **实例弹出 (Instance Pop)**: 遇到 `ENTRYPOINT_SENTINEL` 时，通过 `bvh_instance_pop()` 恢复世界空间光线参数

### 运动模糊图元时间过滤

对于运动模糊曲线和点云，在交叉测试前检查图元的有效时间范围 `prim_time`，若光线时间 `ray->time` 不在范围内则跳过。

### 阴影光线优化

不透明阴影光线（`visibility & PATH_RAY_SHADOW_OPAQUE`）在找到任意交叉点后立即返回 `true`，无需继续搜索最近交叉点。

### GPU/CPU 差异化编译

- GPU: 使用 `ccl_device_inline` 修饰符，将主遍历函数内联以减少调用开销
- CPU: 使用 `ccl_device`（非内联），因为 CPU 分支预测较好且函数体较大

### 自交叉避免

通过 `intersection_skip_self_shadow()` 检查命中图元是否为光线的发射源图元或光源图元，避免自交叉伪影。启用 `__SHADOW_LINKING__` 时还通过 `intersection_skip_shadow_link()` 跳过不在阴影集合中的物体。

## 关联文件

| 文件 | 关系 |
|---|---|
| `kernel/bvh/bvh.h` | 包含本文件生成遍历函数变体 |
| `kernel/bvh/local.h` | 结构类似的局部遍历模板 |
| `kernel/bvh/shadow_all.h` | 结构类似的阴影遍历模板 |
| `kernel/bvh/volume.h` | 结构类似的体积遍历模板 |
| `kernel/bvh/nodes.h` | 节点交叉测试函数 |
| `kernel/bvh/types.h` | 宏和常量定义 |
