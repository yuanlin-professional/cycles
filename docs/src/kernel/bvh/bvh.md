# bvh.h - 层次包围体(BVH)光线追踪场景交叉主入口

## 概述

`bvh.h` 是 Cycles 渲染器中层次包围体(BVH)光线追踪子系统的顶层调度文件。该文件负责根据当前设备类型（CPU/Embree、Metal RT、OptiX、HIP RT）选择对应的硬件加速结构，并在缺乏原生加速结构时回退到软件 BVH2 遍历实现。它通过宏模板机制（`#define BVH_FUNCTION_NAME` + `#include`）编译出针对不同图元特性（毛发、点云、运动模糊）组合的优化遍历变体，并提供统一的场景交叉接口函数供积分器调用。

## 类与结构体

本文件不定义新的类或结构体，但使用了以下关键数据结构：

- **`Ray`**: 光线数据，包含起点 `P`、方向 `D`、时间 `time`、最小/最大距离 `tmin`/`tmax` 等。
- **`Intersection`**: 单次交叉结果，包含距离 `t`、UV 坐标 `u`/`v`、图元索引 `prim`、物体索引 `object`。
- **`LocalIntersection`**: 局部交叉结果（用于次表面散射/倒角），包含命中次数 `num_hits` 及多次交叉记录。
- **`IntegratorShadowState`**: 积分器阴影状态，用于记录透明阴影的多次交叉信息。

## 枚举与常量

本文件通过 `#define` 控制编译变体：

| 宏名 | 作用 |
|---|---|
| `__BVH2__` | 启用软件 BVH2 遍历（无原生加速结构时使用） |
| `__EMBREE__` | 启用 Intel Embree 加速结构 |
| `__METALRT__` | 启用 Apple Metal Ray Tracing |
| `__KERNEL_OPTIX__` | 启用 NVIDIA OptiX 加速结构 |
| `__HIPRT__` | 启用 AMD HIP RT 加速结构 |
| `BVH_HAIR` | 编译毛发曲线交叉支持 |
| `BVH_MOTION` | 编译运动模糊交叉支持 |
| `BVH_POINTCLOUD` | 编译点云交叉支持 |
| `IF_USING_EMBREE` / `IF_NOT_USING_EMBREE` | OneAPI 上 Embree GPU 的特化常量条件分支 |

## 核心函数

### scene_intersect()
- **签名**: `ccl_device_intersect bool scene_intersect(KernelGlobals kg, const ccl_private Ray *ray, const uint visibility, ccl_private Intersection *isect)`
- **功能**: 场景中最近交叉点检测的统一入口。首先验证光线有效性，然后根据设备类型分发到 Embree 或 BVH2 遍历。BVH2 路径根据是否存在运动模糊 (`have_motion`) 和毛发 (`have_curves`) 选择对应的优化变体（`bvh_intersect`、`bvh_intersect_hair`、`bvh_intersect_motion`、`bvh_intersect_hair_motion`）。

### scene_intersect_shadow()
- **签名**: `ccl_device_intersect bool scene_intersect_shadow(KernelGlobals kg, const ccl_private Ray *ray, const uint visibility)`
- **功能**: 不透明阴影光线交叉检测。内部委托给 `scene_intersect()`，仅需判断是否存在遮挡。

### scene_intersect_local()
- **签名**: `template<bool single_hit> ccl_device_intersect bool scene_intersect_local(KernelGlobals kg, const ccl_private Ray *ray, ccl_private LocalIntersection *local_isect, const int local_object, ccl_private uint *lcg_state, const int max_hits)`
- **功能**: 单物体局部 BVH 遍历入口，用于次表面散射（SSS）、环境光遮蔽（AO）和倒角（Bevel）。只在指定物体的 BVH 内进行遍历，支持随机多次命中采样。

### scene_intersect_shadow_all()
- **签名**: `ccl_device_intersect bool scene_intersect_shadow_all(KernelGlobals kg, IntegratorShadowState state, const ccl_private Ray *ray, const uint visibility, const uint max_transparent_hits, ccl_private uint *num_recorded_hits, ccl_private float *throughput)`
- **功能**: 透明阴影光线遍历入口，记录光线路径上的所有交叉点（用于透明阴影计算）。记录命中数和透射率衰减。

### scene_intersect_volume() (单次命中)
- **签名**: `ccl_device_intersect bool scene_intersect_volume(KernelGlobals kg, const ccl_private Ray *ray, ccl_private Intersection *isect, const uint visibility)`
- **功能**: 体积 BVH 遍历入口（单次命中模式），用于初始化或更新体积栈。仅在 `__VOLUME__` 且非 `__VOLUME_RECORD_ALL__` 时编译。

### scene_intersect_volume() (多次命中)
- **签名**: `ccl_device_intersect uint scene_intersect_volume(KernelGlobals kg, const ccl_private Ray *ray, ccl_private Intersection *isect, const uint max_hits, const uint visibility)`
- **功能**: 体积 BVH 遍历入口（多次命中模式），记录最多 `max_hits` 个交叉点。仅在 `__VOLUME__` 且 `__VOLUME_RECORD_ALL__` 时编译。返回实际命中数量。

## 依赖关系

### 内部头文件
- `kernel/bvh/nodes.h` - BVH 节点交叉测试函数
- `kernel/bvh/types.h` - BVH 常量与宏定义
- `kernel/bvh/util.h` - BVH 工具函数（光线偏移、交叉排序等）
- `kernel/bvh/traversal.h` - 常规 BVH2 遍历模板（通过多次 `#include` 生成变体）
- `kernel/bvh/local.h` - 局部 BVH2 遍历模板
- `kernel/bvh/shadow_all.h` - 透明阴影遍历模板
- `kernel/bvh/volume.h` - 体积遍历模板（单次命中）
- `kernel/bvh/volume_all.h` - 体积遍历模板（多次命中）
- `kernel/geom/curve_intersect.h` - 曲线图元交叉
- `kernel/geom/motion_triangle_intersect.h` - 运动三角形交叉
- `kernel/geom/object.h` - 物体变换（实例 push/pop）
- `kernel/geom/point_intersect.h` - 点图元交叉
- `kernel/geom/triangle_intersect.h` - 三角形交叉
- 设备特定加速结构：`kernel/device/cpu/bvh.h`、`kernel/device/metal/bvh.h`、`kernel/device/optix/bvh.h`、`kernel/device/hiprt/bvh.h`

### 被引用
- `src/kernel/integrator/intersect_closest.h` - 最近交叉积分器内核
- `src/kernel/integrator/intersect_dedicated_light.h` - 专用光源交叉
- `src/kernel/integrator/intersect_shadow.h` - 阴影光线交叉
- `src/kernel/integrator/intersect_volume_stack.h` - 体积栈交叉
- `src/kernel/integrator/subsurface_disk.h` - 次表面散射（盘采样）
- `src/kernel/integrator/subsurface_random_walk.h` - 次表面散射（随机游走）
- `src/kernel/svm/ao.h` - 环境光遮蔽节点
- `src/kernel/svm/bevel.h` - 倒角节点
- `src/kernel/osl/services.cpp` - OSL 着色器服务

## 实现细节 / 关键算法

### 宏模板编译策略

由于 GPU 内核语言（OpenCL / CUDA / Metal）不完全支持 C++ 模板，本文件采用预处理器宏模板替代方案：
1. 定义 `BVH_FUNCTION_NAME`（函数名后缀）和 `BVH_FUNCTION_FEATURES`（特性位掩码）
2. `#include` 对应的遍历模板文件（如 `traversal.h`）
3. 模板文件内部通过 `BVH_FUNCTION_FULL_NAME` 宏拼接出唯一函数名
4. 模板文件末尾 `#undef` 这些宏，为下次包含做准备

例如，常规遍历生成了四个变体：
- `bvh_intersect` - 仅点云
- `bvh_intersect_hair` - 毛发+点云
- `bvh_intersect_motion` - 运动模糊+点云
- `bvh_intersect_hair_motion` - 毛发+运动模糊+点云

### 设备分发逻辑

`scene_intersect` 系列函数采用分层分发：
1. 优先检查 Embree（含 OneAPI GPU 特化常量判断）
2. 回退到 BVH2 软件遍历，根据运行时 `kernel_data.bvh` 标志选择最优变体

### OneAPI Embree GPU 特化

在 OneAPI + Embree GPU 配置下，使用 `sycl::specialization_id<RTCFeatureFlags>` 实现编译时分支，允许 AOT（提前编译）二进制同时包含 Embree 和非 Embree 内核路径。

## 关联文件

| 文件 | 关系 |
|---|---|
| `kernel/bvh/types.h` | 提供 BVH 常量和宏定义 |
| `kernel/bvh/nodes.h` | 提供节点交叉测试函数 |
| `kernel/bvh/util.h` | 提供工具函数 |
| `kernel/bvh/traversal.h` | 常规遍历模板 |
| `kernel/bvh/local.h` | 局部遍历模板 |
| `kernel/bvh/shadow_all.h` | 透明阴影遍历模板 |
| `kernel/bvh/volume.h` | 体积遍历模板 |
| `kernel/bvh/volume_all.h` | 体积遍历模板（多次命中） |
| `kernel/device/*/bvh.h` | 各设备的硬件加速实现 |
