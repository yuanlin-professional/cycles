# bvh.h - Metal 光线追踪加速结构(MetalRT)场景求交实现

## 概述

本文件实现了基于 Apple MetalRT 的光线-场景求交功能。它利用 Metal 的硬件光线追踪 API 进行射线与三角形、曲线（毛发）和点云的求交测试，是 Cycles 渲染器在 Metal 设备上进行层次包围体(BVH)遍历的核心模块。

文件定义了多种求交载荷（Payload）结构体，并实现了场景求交、阴影求交、局部求交、全阴影记录求交和体积求交等完整的射线追踪接口。

## 核心函数/宏定义

### 载荷结构体

| 结构体 | 用途 |
|--------|------|
| `MetalRTIntersectionPayload` | 标准场景求交载荷，包含自身图元ID、对象ID和可见性掩码 |
| `MetalRTIntersectionLocalPayload_single_hit` | 局部单次命中求交载荷（用于次表面散射等） |
| `MetalRTIntersectionLocalPayload` | 局部多次命中求交载荷，支持最多 `LOCAL_MAX_HITS` 次命中记录 |
| `MetalRTIntersectionShadowPayload` | 阴影射线求交载荷 |
| `MetalRTIntersectionShadowAllPayload` | 全阴影记录求交载荷，支持透明阴影和吞吐量追踪 |

### 曲线辅助函数

| 函数 | 说明 |
|------|------|
| `curve_ribbon_accept()` | 判断曲线带状（Ribbon）求交是否应被接受，通过比较射线距离与曲线半径来避免自相交 |
| `curve_ribbon_v()` | 计算曲线带状求交的 v 参数坐标，使用 Catmull-Rom 插值获取曲线位置和切线 |

### 场景求交函数

| 函数 | 说明 |
|------|------|
| `scene_intersect()` | 主求交函数，查找射线与场景最近交点，支持三角形、曲线和点云 |
| `scene_intersect_shadow()` | 阴影射线求交，启用 `accept_any_intersection` 以尽早终止 |
| `scene_intersect_local()` | 局部求交（模板函数），用于次表面散射等局部效果，支持单次/多次命中模式 |
| `scene_intersect_shadow_all()` | 全阴影记录求交，记录所有透明阴影命中以计算累计透射率 |
| `scene_intersect_volume()` | 体积求交，剔除曲线和包围盒几何体，仅检测具有体积属性的三角形 |

## 依赖关系

- **内部头文件**:
  - `kernel/bvh/types.h` — 层次包围体(BVH)类型定义
  - `kernel/bvh/util.h` — 层次包围体(BVH)工具函数
- **被引用**:
  - `kernel/bvh/bvh.h` — 当定义 `__METALRT__` 时包含本文件
  - `src/bvh/metal.mm` — Metal 设备侧BVH构建

## 实现细节 / 关键算法

### MetalRT 求交流程

1. 构造 `metal::raytracing::ray` 对象
2. 配置 `metalrt_intersector_type` 求交器，设置不透明度模式为 `non_opaque`（以触发自定义求交测试函数）
3. 根据场景数据动态设置支持的几何体类型（三角形 + 可选的曲线 + 可选的包围盒）
4. 填充载荷结构体（自身图元信息、可见性等）
5. 计算射线掩码 `ray_mask`，映射为 8 位可见性标志
6. 调用 `metalrt_intersect.intersect()` 执行硬件加速求交
7. 根据交点类型（三角形/曲线/包围盒）提取并填充 `Intersection` 结果

### 运动模糊支持

通过 `__METALRT_MOTION__` 宏控制。当启用时，求交调用会传入 `ray->time` 参数，并使用支持运动的加速结构标签（`instance_motion`, `primitive_motion`）。

### 局部求交的优化策略

- 非运动模糊时，直接从底层加速结构（BLAS）遍历，避免顶层加速结构开销
- 单次命中模式可设置为 `opaque` 以跳过自定义过滤（当不需要自相交检测时）
- 多次命中模式使用 LCG 随机数进行蓄水池采样，在命中数超过上限时随机替换已有记录

### 射线掩码处理

```
uint ray_mask = visibility & 0xFF;
if (0 == ray_mask && (visibility & ~0xFF) != 0) {
    ray_mask = 0xFF;
}
```
将 Cycles 的可见性标志映射为 MetalRT 的 8 位射线掩码。当高位有值但低 8 位为 0 时，设为全通过。

## 关联文件

- `kernel/device/metal/compat.h` — Metal 兼容层，定义 MetalRT 类型别名
- `kernel/device/metal/globals.h` — Metal 全局数据结构
- `kernel/device/metal/kernel.metal` — Metal 内核入口点，包含 MetalRT 求交处理函数
- `kernel/device/gpu/kernel.h` — GPU 通用内核框架
- `kernel/bvh/bvh.h` — BVH 遍历调度入口
