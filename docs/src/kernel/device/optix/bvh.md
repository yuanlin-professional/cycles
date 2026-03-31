# bvh.h - OptiX 光线追踪场景求交与命中处理实现

## 概述

本文件实现了基于 NVIDIA OptiX 光线追踪引擎的完整场景求交接口。它包含两大部分：一是 OptiX 程序组回调函数（miss/anyhit/closesthit/intersection），二是供积分器调用的场景求交函数。OptiX 通过着色器绑定表（SBT）和光线载荷（payload）寄存器机制将数据在主机代码与设备回调之间传递。

## 核心函数/宏定义

### 工具函数

- **`get_payload_ptr_0<T>()`** / **`get_payload_ptr_2<T>()`** / **`get_payload_ptr_6<T>()`** - 从 OptiX payload 寄存器中解包指针。使用 `pointer_unpack_from_uint` 将两个 32 位寄存器重组为 64 位设备指针。
- **`get_object_id()`** - 获取对象 ID。在启用 `__OBJECT_MOTION__` 时从 TLAS 变换列表句柄获取实例 ID（跳过可能存在的运动变换节点），否则直接使用 `optixGetInstanceId()`。

### OptiX 程序组回调

- **`__miss__kernel_optix_miss()`** - 未命中回调。设置 payload 的交点距离为 `tmax`，图元类型为 `PRIMITIVE_NONE`。
- **`__anyhit__kernel_optix_ignore()`** - 忽略命中的 any-hit 回调，调用 `optixIgnoreIntersection()`。
- **`__closesthit__kernel_optix_ignore()`** - 空的 closest-hit 回调。
- **`__anyhit__kernel_optix_local_hit()`** - 局部求交 any-hit 回调，用于子表面散射。过滤非三角形图元，验证对象匹配，执行水库采样记录多个命中，计算几何法线。
- **`__anyhit__kernel_optix_shadow_all_hit()`** - 阴影全记录 any-hit 回调。处理三角形、曲线和点云的命中，记录透明交点到积分器状态数组，处理曲线阴影透明度衰减。
- **`__anyhit__kernel_optix_volume_test()`** - 体积测试 any-hit 回调。忽略非三角形图元和无体积属性的对象。
- **`__anyhit__kernel_optix_visibility_test()`** - 可见性测试 any-hit 回调。处理不透明阴影的提前终止（`optixTerminateRay`）和自相交排除。
- **`__closesthit__kernel_optix_hit()`** - 最近命中回调。将命中信息（距离、重心坐标、图元索引、对象 ID、图元类型）写入 payload 寄存器，分别处理三角形、曲线和点云。

### 自定义图元求交

- **`optix_intersection_curve(prim, type)`** - 曲线求交实现，调用 `curve_intersect()` 并通过 `optixReportIntersection` 报告命中。
- **`__intersection__curve_ribbon()`** - 曲线带状图元求交入口，过滤非 ribbon 类型曲线。
- **`__intersection__point()`** - 点云图元求交入口，调用 `point_intersect()`。

### 场景求交函数

- **`scene_intersect()`** - 最近交点求交。将 8 个 payload 寄存器编码光线数据和可见性标志，调用 `optixTrace`，从返回的 payload 解码交点信息。
- **`scene_intersect_shadow()`** - 阴影求交。使用 `optixTraverse` + `optixHitObjectIsHit` 进行求交测试。
- **`scene_intersect_local()`** - 局部求交（子表面散射），使用 SBT 偏移 2（`PG_HITL`）路由到 `__anyhit__kernel_optix_local_hit`。
- **`scene_intersect_shadow_all()`** - 阴影全记录求交，使用 SBT 偏移 1（`PG_HITS`）路由到 `__anyhit__kernel_optix_shadow_all_hit`。
- **`scene_intersect_volume()`** - 体积求交，使用 SBT 偏移 3（`PG_HITV`）路由到 `__anyhit__kernel_optix_volume_test`。

## 依赖关系

- **内部头文件**:
  - `kernel/bvh/types.h` - BVH 类型定义
  - `kernel/bvh/util.h` - BVH 工具函数
  - `<optix_function_table.h>` - OptiX ABI 版本定义（仅定义 ABI 版本）

- **被引用**:
  - `kernel/bvh/bvh.h` - BVH 抽象层根据编译后端条件引入本文件

## 实现细节 / 关键算法

1. **Payload 寄存器编码**: OptiX 提供 8 个 32 位 payload 寄存器（p0-p7）。64 位指针通过拆分为两个 32 位值编码到寄存器对中（如 p6/p7 存储 Ray 指针）。不同的求交类型对寄存器有不同的编码约定。

2. **SBT 偏移路由**: 通过 `optixTrace`/`optixTraverse` 的 SBT 偏移参数控制 OptiX 调用哪组命中程序：偏移 0 = 默认最近命中（`PG_HITD`）；偏移 1 = 阴影全记录（`PG_HITS`）；偏移 2 = 局部命中（`PG_HITL`）；偏移 3 = 体积命中（`PG_HITV`）。

3. **光线掩码与标志**: `ray_mask` 取 `visibility` 的低 8 位用于 OptiX 实例掩码匹配。当低 8 位为 0 但高位非零时，使用 0xFF 全通过掩码。对不透明阴影光线追加 `OPTIX_RAY_FLAG_TERMINATE_ON_FIRST_HIT` 以提前终止。

4. **水库采样**: `__anyhit__kernel_optix_local_hit` 中使用 LCG 随机数进行水库采样，在有限的命中槽位中均匀采样所有可能的命中点，用于子表面散射的多点采样。

5. **optixTrace vs optixTraverse**: `scene_intersect` 使用 `optixTrace`（完整遍历+着色），`scene_intersect_shadow` 使用 `optixTraverse`（仅遍历）+ `optixHitObjectIsHit`（查询命中状态），后者避免了不必要的着色调用开销。

## 关联文件

- `kernel/device/optix/compat.h` - OptiX 兼容层
- `kernel/device/optix/globals.h` - OptiX 全局参数
- `kernel/device/optix/kernel.cu` - 主内核文件（间接包含本文件）
- `kernel/bvh/bvh.h` - BVH 抽象层入口
- `kernel/geom/curve_intersect.h` - 曲线求交算法
- `kernel/geom/point_intersect.h` - 点云求交算法
