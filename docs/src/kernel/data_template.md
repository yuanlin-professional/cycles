# data_template.h - 内核常量数据结构模板定义

## 概述

`data_template.h` 是 Cycles 内核常量数据（`KernelData` 中的可特化成员）的模板定义文件。它通过 `KERNEL_STRUCT_BEGIN`、`KERNEL_STRUCT_MEMBER`、`KERNEL_STRUCT_END` 等宏，以声明式方式定义了背景、层次包围体(BVH)、胶片/成像平面、积分器以及着色器虚拟机(SVM)用量等核心配置结构体。与 `data_arrays.h` 类似，本文件作为 X-Macro 模板被多次 `#include`，可在不同上下文中展开为结构体定义、GPU 常量声明或特化常量等。

## 类与结构体

### KernelBackground（背景配置）
场景背景/环境光相关参数：
- `sun` (`float4`) — 太阳方向（xyz）和角度（w），使用 float4 确保跨设备对齐
- `use_sun_guiding` (`int`) — 是否启用太阳引导采样
- `surface_shader` / `volume_shader` (`int`) — 表面/体积着色器索引
- `transparent` (`int`) — 是否透明背景
- `transparent_roughness_squared_threshold` (`float`) — 透明粗糙度阈值
- `sun_weight` / `map_weight` / `portal_weight` (`float`) — 太阳/重要性图/门户采样权重
- `map_res_x` / `map_res_y` (`int`) — 重要性图分辨率
- `use_mis` (`int`) — 是否启用多重重要性采样(MIS)
- `lightgroup` (`int`) — 光组索引
- `light_index` / `object_index` (`int`) — 光源/对象索引

### KernelBVH（层次包围体配置）
BVH2 加速结构参数（非原生硬件加速时使用）：
- `root` (`int`) — 根节点索引
- `have_motion` / `have_curves` / `have_points` / `have_volumes` (`int`) — 场景中各类几何体存在标志
- `bvh_layout` (`int`) — BVH 布局类型
- `use_bvh_steps` (`int`) — 是否使用 BVH 步进
- `curve_subdivisions` (`int`) — 曲线细分级别

### KernelFilm（胶片/成像平面配置）
渲染输出和通道相关的完整配置，主要包括：
- 色彩空间变换矩阵: `xyz_to_r/g/b`, `rgb_to_y`, `white_xyz`, `rec709_to_r/g/b`
- 曝光: `exposure`
- **渲染通道偏移**: 大量 `pass_*` 成员，定义了各渲染通道（深度、法线、运动向量、漫反射、光泽反射、透射、体积、AO、Cryptomatte、去噪、AOV、光组、烘焙、路径引导等）在输出缓冲区中的偏移
- 雾效参数: `mist_start`, `mist_inv_depth`, `mist_falloff`
- 阴影捕捉: `use_approximate_shadow_catcher`

### KernelIntegrator（积分器配置）
路径追踪积分器的核心参数：
- **光源采样**: `use_direct_light`, `use_light_mis`, `use_light_tree`, `num_lights`, `num_distant_lights`, `num_background_lights`
- **门户采样**: `num_portals`, `portal_offset`
- **光源分布**: `num_distribution`, `distribution_pdf_triangles`, `distribution_pdf_lights`
- **光线弹射**: `min_bounce`, `max_bounce`, `max_diffuse_bounce`, `max_glossy_bounce`, `max_transmission_bounce`, `max_volume_bounce`
- **AO 弹射**: `ao_bounces`, `ao_bounces_distance`, `ao_bounces_factor`
- **透明度**: `transparent_min_bounce`, `transparent_max_bounce`
- **焦散**: `caustics_reflective`, `caustics_refractive`, `filter_glossy`
- **采样**: `seed`, `sampling_pattern`, `scrambling_distance`
- **Sobol 序列**: `tabulated_sobol_sequence_size`, `sobol_index_mask`, `blue_noise_sequence_length`
- **体积渲染**: `use_volumes`, `volume_ray_marching`, `volume_max_steps`
- **路径引导**: `surface_guiding_probability`, `volume_guiding_probability`, `guiding_distribution_type`, `guiding_roughness_threshold` 等

### KernelSVMUsage（着色器虚拟机用量）
通过 `#include "kernel/svm/node_types_template.h"` 自动为每种 SVM 节点类型生成一个 `int` 成员，用于着色器特化编译时标记哪些节点类型被实际使用。

## 枚举与常量

无独立枚举。部分成员使用 `KERNEL_STRUCT_MEMBER_DONT_SPECIALIZE` 标记，表示该成员在内核特化编译时不应被常量替换（如 `seed` 每帧变化）。

## 核心函数

无函数定义。本文件是纯粹的宏驱动声明模板。

## 依赖关系

- **内部头文件**: `kernel/svm/node_types_template.h`（被 KernelSVMUsage 段间接 include）
- **被引用**:
  - `kernel/types.h` — 两次 include 本文件：第一次定义结构体，第二次在 `KernelData` 内部实例化成员
  - `kernel/device/metal/function_constants.h` — Metal 设备端函数常量特化
  - `device/metal/device_impl.mm` / `device/metal/kernel.mm` — Metal 后端实现

## 实现细节 / 关键算法

本文件同样采用 **X-Macro 模式**。调用方在 include 前定义三个宏：
- `KERNEL_STRUCT_BEGIN(name, parent)` — 结构体开始
- `KERNEL_STRUCT_MEMBER(parent, type, name)` — 结构体成员
- `KERNEL_STRUCT_END(name)` — 结构体结束

在 `types.h` 中本文件被 include 两次，用途不同：
1. **第一次**（约第 1446 行）：将宏展开为 C 结构体定义（如 `struct KernelBackground { ... };`）
2. **第二次**（约第 1491 行）：在 `KernelData` 结构体内部展开为成员实例化（如 `KernelBackground background;`）

在支持常量特化的设备端（如 Metal），`KERNEL_STRUCT_MEMBER` 可被重定义为生成设备端常量，从而实现编译期内核特化优化。`KERNEL_STRUCT_MEMBER_DONT_SPECIALIZE` 标记的成员（如随机种子）会被排除在特化之外。

## 关联文件

- `kernel/types.h` — 主要消费者，两次 include 本文件以生成结构体和 KernelData 成员
- `kernel/data_arrays.h` — 姊妹模板文件，定义数组类型数据
- `kernel/svm/node_types_template.h` — 提供 SVM 节点类型列表
- `kernel/device/metal/function_constants.h` — Metal 设备端常量特化实现
