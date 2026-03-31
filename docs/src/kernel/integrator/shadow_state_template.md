# shadow_state_template.h - 阴影路径状态数据模板

## 概述

`shadow_state_template.h` 使用宏模板系统定义了阴影路径的全部状态数据结构。这不是传统的头文件，而是通过预处理器宏 `KERNEL_STRUCT_*` 在不同的上下文中被多次包含，每次生成不同的代码（CPU 的 AoS 结构体、GPU 的 SoA 数组、状态复制代码等）。该文件定义了阴影光线追踪所需的所有中间状态。

## 核心函数

本文件不包含函数，而是定义以下数据结构：

### `shadow_path` 结构
阴影路径的核心状态，包含：
- **像素与采样**: `render_pixel_index`, `sample` - 目标像素和当前采样编号
- **随机数**: `rng_pixel`, `rng_offset` - 每像素随机数种子与维度偏移
- **弹射深度**: `bounce`, `transparent_bounce`, `diffuse_bounce`, `glossy_bounce`, `transmission_bounce`, `volume_bounds_bounce`, `portal_bounce`
- **内核调度**: `queued_kernel` - 下一个待执行的设备内核
- **路径标志**: `flag` - `PathRayFlag` 枚举值
- **光谱数据**: `throughput`（吞吐量）, `unshadowed_throughput`（未遮挡吞吐量，用于AO）, `pass_diffuse_weight`/`pass_glossy_weight`（渲染通道权重）
- **交点信息**: `num_hits` - 阴影光线找到的交点总数
- **灯光分组**: `lightgroup` - 灯光组标识
- **路径引导**: `unlit_throughput`, `path_segment`, `guiding_mis_weight`

### `shadow_ray` 结构（紧凑布局）
阴影光线参数：`P`(起点), `D`(方向), `tmin`, `tmax`, `time`, `dP`(微分), `self_light_object`, `self_light_prim`。

### `shadow_isect` 数组结构
阴影交点结果，存储 `INTEGRATOR_SHADOW_ISECT_SIZE` 个最近交点：每个包含 `t`, `u`, `v`, `prim`, `object`, `type`。CPU 和 GPU 使用不同的数组大小。

### `shadow_volume_stack` 数组结构
阴影体积栈：每个条目包含 `object` 和 `shader`，大小为 `KERNEL_STRUCT_VOLUME_STACK_SIZE`。

## 依赖关系

- **内部头文件**: 无直接 #include（通过宏包含机制使用）
- **被引用**: `state.h`（生成 CPU `IntegratorShadowStateCPU` 和 GPU `IntegratorStateGPU` 结构）、`state_util.h`（生成状态复制代码）、`src/integrator/path_trace_work_gpu.cpp`（GPU 内存分配）、`kernel/types.h`

## 实现细节 / 关键算法

1. **宏模板系统**: 文件使用 `KERNEL_STRUCT_BEGIN`/`KERNEL_STRUCT_MEMBER`/`KERNEL_STRUCT_END` 等宏，在不同包含上下文中展开为不同代码。例如在 `state.h` 中展开为结构体成员定义，在 `state_util.h` 中展开为逐字段复制代码。

2. **特性标志条件编译**: 每个成员都关联一个特性标志（如 `KERNEL_FEATURE_PATH_TRACING`、`KERNEL_FEATURE_VOLUME`），使编译器可以在不需要某些特性时跳过对应字段，节省内存。

3. **CPU/GPU 数组大小差异**: `shadow_isect` 在 CPU 和 GPU 上使用不同的数组大小（`INTEGRATOR_SHADOW_ISECT_SIZE_CPU` vs `INTEGRATOR_SHADOW_ISECT_SIZE_GPU`），GPU 通常使用更小的大小以节省显存。

4. **紧凑打包结构**: `shadow_ray` 使用 `KERNEL_STRUCT_BEGIN_PACKED` 和 `KERNEL_STRUCT_MEMBER_PACKED` 宏，在 GPU 上可进行打包优化以减少内存读写次数。

## 关联文件

- `state_template.h` - 主路径状态的对应模板
- `state.h` - 消费模板生成最终数据结构
- `state_util.h` - 消费模板生成状态读写和复制工具函数
