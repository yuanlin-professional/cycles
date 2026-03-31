# util.h - BVH 光线追踪工具函数集

## 概述

`util.h` 是 BVH 子系统的工具函数集合，提供了光线有效性验证、自交叉避免、交叉距离偏移、光线偏移、交叉结果排序、着色器标志查询、属性查找、曲线阴影透明度计算以及阴影链接过滤等核心辅助功能。该文件在光线追踪管线的多个阶段被广泛使用，不仅服务于 BVH 遍历本身，还被积分器、着色器和几何模块引用。

## 类与结构体

本文件不定义新的类或结构体，但密切使用：

- **`Ray`**: 光线数据，使用其 `P`（起点）、`D`（方向）、`self`（自交叉信息）成员。
- **`RaySelfPrimitives`**: 光线自身图元信息，包含 `prim`、`object`、`light_prim`、`light_object`，用于自交叉避免。
- **`Intersection`**: 交叉结果，使用其 `t`（距离）、`prim`、`object`、`type`、`u` 成员。
- **`KernelCurve`**: 曲线内核数据。
- **`AttributeMap`**: 属性映射表条目。
- **`IntegratorShadowState`**: 积分器阴影状态。

## 枚举与常量

| 常量 | 值 | 说明 |
|---|---|---|
| `CURVE_SHADOW_TRANSPARENCY_CUTOFF` | `0.001f` | 曲线阴影透明度截止值，低于此值视为完全不透明 |
| `ATTR_STD_SHADOW_TRANSPARENCY` | - | 阴影透明度属性标准 ID |
| `ATTR_STD_NONE` | - | 属性表中的空条目/跳转标记 |
| `ATTR_STD_NOT_FOUND` | - | 属性未找到返回值 |
| `ATTR_PRIM_TYPES` | - | 属性表中每个属性占用的条目数 |
| `LIGHT_LINK_MASK_ALL` | - | 所有阴影集合的掩码（默认值） |

## 核心函数

### intersection_ray_valid()
- **签名**: `ccl_device_inline bool intersection_ray_valid(const ccl_private Ray *ray)`
- **功能**: 验证光线参数是否有效。检查起点 `P.x` 和方向 `D.x` 是否为有限值，且方向向量长度非零。防止无效光线（如 NaN）导致 BVH 遍历栈溢出。也过滤条件追踪中求值为 false 的空光线。

### intersection_t_offset()
- **签名**: `ccl_device_forceinline float intersection_t_offset(const float t)`
- **功能**: 返回距离 `t` 的最小可表示浮点增量（简化版 `nextafterf`），用于跳过当前交叉距离。对零值特殊处理返回 `FLT_MIN`（最小归一化浮点数），避免非归一化零值导致的假阳性交叉。

### ray_offset()
- **签名**: `ccl_device_inline float3 ray_offset(const float3 P, const float3 Ng)`
- **功能**: 计算经过偏移的光线起点以避免自交叉。实现来自 "A Fast and Robust Method for Avoiding Self-Intersection"（Ray Tracing Gems 第 6 章）。在 ULP（最小精度单位）级别沿法线方向偏移位置，对接近原点的位置使用固定偏移量。

### intersections_compare()
- **签名**: `ccl_device int intersections_compare(const void *a, const void *b)`
- **功能**: 交叉结果比较函数（仅 CPU），可与 `qsort` 配合使用，按距离 `t` 升序排列。

### sort_intersections_and_normals()
- **签名**: `ccl_device_inline void sort_intersections_and_normals(ccl_private Intersection *hits, ccl_private float3 *Ng, uint num_hits)`
- **功能**: 冒泡排序交叉结果及其对应法线，按距离 `t` 升序排列。用于次表面散射等场景中少量交叉的排序（冒泡排序对小数据量在 CPU 和 GPU 上均适用）。

### intersection_get_shader_flags()
- **签名**: `ccl_device_forceinline int intersection_get_shader_flags(KernelGlobals kg, const int prim, const int type)`
- **功能**: 根据图元类型（三角形/点/曲线）获取对应着色器的标志位。用于快速判断材质属性（如是否有透明阴影、是否仅为体积等）。

### intersection_get_shader_from_isect_prim()
- **签名**: `ccl_device_forceinline int intersection_get_shader_from_isect_prim(KernelGlobals kg, const int prim, const int isect_type)`
- **功能**: 从图元和交叉类型获取着色器索引（经 `SHADER_MASK` 掩码）。

### intersection_get_shader()
- **签名**: `ccl_device_forceinline int intersection_get_shader(KernelGlobals kg, const ccl_private Intersection *ccl_restrict isect)`
- **功能**: 从交叉结果获取着色器索引的便捷包装。

### intersection_get_object_flags()
- **签名**: `ccl_device_forceinline int intersection_get_object_flags(KernelGlobals kg, const ccl_private Intersection *ccl_restrict isect)`
- **功能**: 获取交叉物体的标志位。

### intersection_find_attribute()
- **签名**: `ccl_device_inline int intersection_find_attribute(KernelGlobals kg, const int object, const uint id)`
- **功能**: 在物体的属性映射表中查找指定 ID 的属性偏移。支持链式跳转（当表分段存储时通过 `ATTR_STD_NONE` + `element == 0` 标记跳转到新偏移）。返回属性数据偏移或 `ATTR_STD_NOT_FOUND`。

### intersection_curve_shadow_transparency()
- **签名**: `ccl_device_inline float intersection_curve_shadow_transparency(KernelGlobals kg, const int object, const int prim, const int type, const float u)`
- **功能**: 获取曲线图元在交叉点处的阴影透明度值。通过查找 `ATTR_STD_SHADOW_TRANSPARENCY` 属性，在曲线的两个控制点 `k0`/`k1` 之间线性插值。未找到属性时返回 0.0（不透明）。

### intersection_skip_self()
- **签名**: `ccl_device_inline bool intersection_skip_self(const ccl_ray_data RaySelfPrimitives &self, const int object, const int prim)`
- **功能**: 检查交叉图元是否为光线的发射源图元（自交叉检测）。

### intersection_skip_self_shadow()
- **签名**: `ccl_device_inline bool intersection_skip_self_shadow(const ccl_ray_data RaySelfPrimitives &self, const int object, const int prim)`
- **功能**: 扩展的自交叉检测，同时检查发射源图元和光源图元。

### intersection_skip_self_local()
- **签名**: `ccl_device_inline bool intersection_skip_self_local(const ccl_ray_data RaySelfPrimitives &self, const int prim)`
- **功能**: 局部遍历的自交叉检测（仅检查图元 ID，因为物体已确定）。

### intersection_skip_shadow_link()
- **签名**: `ccl_device_inline bool intersection_skip_shadow_link(KernelGlobals kg, const ccl_ray_data RaySelfPrimitives &self, const int isect_object)`
- **功能**: 阴影链接过滤。检查被交叉物体是否在光源的阴影集合成员中，不在则跳过该交叉。使用位掩码 `shadow_set_membership` 和 `blocker_shadow_set` 进行高效检测。

### intersection_skip_shadow_already_recoded()
- **签名**: `ccl_device_forceinline bool intersection_skip_shadow_already_recoded(IntegratorShadowState state, const int object, const int prim, const int num_hits)`
- **功能**: 检查阴影交叉是否已被记录（处理 BVH 空间分裂导致同一图元被多次命中的情况）。遍历已记录的交叉数组进行匹配。

## 依赖关系

### 内部头文件
- `kernel/globals.h` - 内核全局数据
- `kernel/integrator/state.h` - 积分器状态定义（`IntegratorShadowState`）
- `kernel/types.h` - 基础类型定义（`Intersection`、`Ray` 等）

### 被引用
- `src/kernel/bvh/bvh.h` - 直接 `#include "kernel/bvh/util.h"`
- `src/kernel/device/cpu/bvh.h` - CPU/Embree 设备实现
- `src/kernel/device/metal/bvh.h` - Metal 设备实现
- `src/kernel/device/optix/bvh.h` - OptiX 设备实现
- `src/kernel/geom/motion_curve.h` - 运动曲线几何
- `src/kernel/geom/motion_point.h` - 运动点几何
- `src/kernel/geom/motion_triangle.h` - 运动三角形几何

（此外，`util.h` 中的函数被 `traversal.h`、`shadow_all.h`、`local.h`、`volume.h`、`volume_all.h` 等遍历模板以及多个积分器文件间接使用。）

## 实现细节 / 关键算法

### 光线偏移算法（Ray Tracing Gems 第 6 章）

`ray_offset()` 使用整数运算在 ULP 级别偏移浮点坐标：
1. 将法线方向缩放到整数偏移量（`int_scale = 256.0f`）
2. 将浮点位模式视为整数，加上偏移量（利用 IEEE 754 浮点数的单调性：对正数加 1 ULP 等于加上整数 1）
3. 对接近原点的坐标（`|P| < 1/32`）使用固定浮点偏移（`float_scale = 1/65536`），因为 ULP 偏移对极小值不够大

### 属性查找的链式跳转

属性映射表可能被分段存储，`intersection_find_attribute()` 支持通过特殊标记 `(id == ATTR_STD_NONE && element == 0)` 进行跳转到另一段表的起始偏移，实现了类似链表的查找。

### 阴影链接位掩码

`intersection_skip_shadow_link()` 使用 64 位掩码实现高效的集合成员检测：每个阴影集合对应一个位，光源的 `shadow_set_membership` 掩码标记其影响的集合，遮挡物体的 `blocker_shadow_set` 标记其所属集合。通过位与运算即可判断遮挡关系。

## 关联文件

| 文件 | 关系 |
|---|---|
| `kernel/bvh/bvh.h` | 引入本文件 |
| `kernel/bvh/types.h` | 提供 `CURVE_SHADOW_TRANSPARENCY_CUTOFF` 被本文件使用 |
| `kernel/bvh/traversal.h` | 使用本文件的自交叉和阴影工具函数 |
| `kernel/bvh/shadow_all.h` | 使用本文件的阴影相关函数 |
| `kernel/integrator/shade_surface.h` | 使用交叉着色器查询函数 |
| `kernel/integrator/shade_shadow.h` | 使用阴影工具函数 |
| `kernel/light/sample.h` | 使用交叉查询函数 |
