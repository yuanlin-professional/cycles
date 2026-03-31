# local.h - 单物体局部层次包围体(BVH)遍历模板

## 概述

`local.h` 是一个 BVH 遍历模板文件，实现了针对单个物体的局部交叉检测。该文件专门服务于次表面散射（SSS）、环境光遮蔽（AO）和倒角（Bevel）等需要在单个物体表面附近进行多次采样的渲染效果。与全场景遍历不同，局部遍历仅在指定物体的 BVH 子树内进行搜索，并支持随机多次命中记录。该文件通过宏模板机制被 `bvh.h` 多次包含，生成有无运动模糊的优化变体。

## 类与结构体

本文件不定义新的结构体，但使用：

- **`LocalIntersection`**: 局部交叉结果容器，包含 `num_hits`（命中计数）和多条交叉记录。
- **`Ray`**: 光线参数（起点、方向、时间范围等）。

## 枚举与常量

通过宏从 `types.h` 继承：

| 宏 / 常量 | 说明 |
|---|---|
| `BVH_STACK_SIZE` (192) | 遍历栈大小 |
| `ENTRYPOINT_SENTINEL` (0x76543210) | 栈底哨兵值 |
| `BVH_HAIR` | 控制是否使用支持非轴对齐节点的交叉函数 |
| `BVH_MOTION` | 控制是否编译运动模糊图元交叉代码 |
| `NODE_INTERSECT` | 根据 `BVH_HAIR` 特性选择 `bvh_node_intersect` 或 `bvh_aligned_node_intersect` |

## 核心函数

### BVH_FUNCTION_FULL_NAME(BVH)()
- **签名**: `bool BVH_FUNCTION_FULL_NAME(BVH)(KernelGlobals kg, const ccl_private Ray *ray, ccl_private LocalIntersection *local_isect, const int local_object, ccl_private uint *lcg_state, const int max_hits)`
- **功能**: 在指定物体（`local_object`）的 BVH 子树内遍历，查找与光线相交的三角形图元。仅处理三角形（含运动三角形），忽略曲线和点云图元。支持随机采样多次命中（通过 `lcg_state` 线性同余生成器状态）。当 `max_hits` 为 0 时，`local_isect` 可以为 `nullptr`，仅做存在性检测。

### BVH_FUNCTION_NAME()
- **签名**: `bool BVH_FUNCTION_NAME(KernelGlobals kg, const ccl_private Ray *ray, ccl_private LocalIntersection *local_isect, const int local_object, ccl_private uint *lcg_state, const int max_hits)`
- **功能**: 内联包装函数，直接转发到 `BVH_FUNCTION_FULL_NAME(BVH)()`。生成的具体名称为 `bvh_intersect_local` 或 `bvh_intersect_local_motion`。

## 依赖关系

### 内部头文件
本文件作为模板被 `#include`，不自行包含其他头文件。依赖由 `bvh.h` 提前引入的：
- `kernel/bvh/types.h` - 宏定义（`BVH_FEATURE`、`BVH_FUNCTION_FULL_NAME` 等）
- `kernel/bvh/nodes.h` - 节点交叉函数（`bvh_node_intersect`、`bvh_aligned_node_intersect`）
- `kernel/bvh/util.h` - 工具函数（`bvh_clamp_direction`、`bvh_inverse_direction`）
- `kernel/geom/object.h` - 实例变换函数（`bvh_instance_push`、`bvh_instance_motion_push`）
- `kernel/geom/triangle_intersect.h` - `triangle_intersect_local()`
- `kernel/geom/motion_triangle_intersect.h` - `motion_triangle_intersect_local()`

### 被引用
- `src/kernel/bvh/bvh.h` - 通过 `#include "kernel/bvh/local.h"` 两次包含（普通版和运动模糊版）

## 实现细节 / 关键算法

### 局部遍历与全场景遍历的差异

1. **起始节点**: 不从 BVH 根节点开始，而是从目标物体的根节点 `kernel_data_fetch(object_node, local_object)` 开始遍历。
2. **无实例遍历**: 不处理实例节点（instance push/pop），因为已经定位到了具体物体。
3. **物体过滤**: 对于非实例化物体（`SD_OBJECT_TRANSFORM_APPLIED` 未设置），需要在叶节点检查 `prim_object == local_object` 以确保只命中目标物体。
4. **图元限制**: 仅处理 `PRIMITIVE_TRIANGLE` 和 `PRIMITIVE_MOTION_TRIANGLE`，跳过曲线和点云。
5. **自交叉避免**: 通过 `intersection_skip_self_local()` 跳过发射光线的源图元。

### 遍历栈机制

使用线程局部数组 `traversal_stack[BVH_STACK_SIZE]` 实现 BVH 遍历栈：
- 内部节点：测试左右子节点与光线的交叉，将较远子节点压栈，继续遍历较近子节点
- 叶节点（`node_addr < 0`）：遍历叶节点中的图元列表进行精确交叉测试
- 使用双层 `do-while` 循环，外层处理 `ENTRYPOINT_SENTINEL` 哨兵弹出

### 实例变换

如果目标物体未应用变换（非 `SD_OBJECT_TRANSFORM_APPLIED`），在遍历前将光线变换到物体局部空间：
- 无运动模糊时调用 `bvh_instance_push()`
- 有运动模糊时调用 `bvh_instance_motion_push()`

## 关联文件

| 文件 | 关系 |
|---|---|
| `kernel/bvh/bvh.h` | 包含本文件并生成具体遍历函数 |
| `kernel/bvh/types.h` | 提供宏定义和常量 |
| `kernel/bvh/nodes.h` | 提供节点交叉测试 |
| `kernel/bvh/traversal.h` | 相似结构的全场景遍历模板 |
| `kernel/geom/triangle_intersect.h` | 三角形局部交叉实现 |
