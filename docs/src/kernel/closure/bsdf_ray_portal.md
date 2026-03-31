# bsdf_ray_portal.h - 光线传送门双向散射分布函数(BSDF)闭包

## 概述

本文件实现了 Cycles 渲染器中光线传送门（Ray Portal）的双向散射分布函数(BSDF)闭包。光线传送门是一种特殊的传输机制，允许光线从一个空间位置瞬间传送到另一个位置并改变方向，常用于实现传送门特效。该闭包不参与常规的 BSDF 求值（eval 始终返回零），仅通过专用的采样路径触发光线传送。

## 类与结构体

### `RayPortalClosure`
光线传送门闭包的数据结构，继承自 `SHADER_CLOSURE_BASE`。

| 成员 | 类型 | 说明 |
|------|------|------|
| `P` | `float3` | 传送目标位置 |
| `D` | `float3` | 传送后的光线方向 |

编译时通过 `static_assert` 确保 `RayPortalClosure` 不超过 `ShaderClosure` 的大小。

## 核心函数

### `bsdf_ray_portal_setup`
```cpp
ccl_device void bsdf_ray_portal_setup(ccl_private ShaderData *sd,
                                      const Spectrum weight,
                                      const float3 position,
                                      float3 direction)
```
初始化光线传送门闭包。主要流程：
1. 检查权重是否超过 `CLOSURE_WEIGHT_CUTOFF` 截止阈值，低于阈值则直接返回
2. 将权重累加到 `sd->closure_transparent_extinction`（透明消光项）
3. 通过 `closure_alloc` 分配闭包内存，类型为 `CLOSURE_BSDF_RAY_PORTAL_ID`
4. 设置 `SD_BSDF | SD_RAY_PORTAL` 标志位
5. 若传入方向为零向量，则默认使用入射方向的反方向 `-sd->wi`
6. 存储传送目标位置 `P` 和方向 `D`

### `bsdf_ray_portal_eval`
```cpp
ccl_device Spectrum bsdf_ray_portal_eval(...)
```
求值函数。始终返回零光谱值和零概率密度。光线传送门不参与常规的 BSDF 求值，因为它是一种 delta 分布——光线只能沿特定方向传送，无法通过连续采样命中。

## 依赖关系
- **内部头文件**:
  - `kernel/types.h` — 内核类型定义
  - `kernel/closure/alloc.h` — 闭包内存分配
- **被引用**:
  - `src/kernel/closure/bsdf.h` — BSDF 调度总入口

## 实现细节 / 关键算法

- 光线传送门本质上是一个 delta 函数分布的 BSDF，因此 `eval` 始终返回零。实际的传送逻辑在积分器的采样阶段处理，而非在此闭包内。
- 该闭包同时设置 `SD_BSDF` 和 `SD_RAY_PORTAL` 标志，后者用于告知积分器当前着色点存在传送门行为。
- 权重被累加到 `closure_transparent_extinction`，使传送门在视觉上表现为透明区域。

## 关联文件
- `src/kernel/closure/bsdf.h` — BSDF 统一调度，负责调用本文件的 eval 函数
- `src/kernel/closure/alloc.h` — 闭包分配器
- `src/kernel/closure/bsdf_transparent.h` — 透明 BSDF，与传送门有类似的透明消光处理
