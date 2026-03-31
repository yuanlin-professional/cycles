# state_template.h - 主路径状态数据模板

## 概述

`state_template.h` 使用宏模板系统定义了主路径追踪的完整状态数据结构。与 `shadow_state_template.h` 类似，该文件通过预处理器宏在不同上下文中被多次包含以生成不同的代码。它定义了路径追踪的全部中间状态，包括路径参数、光线数据、交点结果、次表面散射参数、体积栈、路径引导状态和阴影链接状态。

## 核心函数

本文件不包含函数，定义以下数据结构模板：

### `path` 结构
路径的核心状态：
- **像素与采样**: `render_pixel_index`, `sample`
- **弹射深度**: `bounce`, `transparent_bounce`, `diffuse_bounce`, `glossy_bounce`, `transmission_bounce`, `volume_bounce`, `volume_bounds_bounce`, `portal_bounce`
- **内核调度**: `queued_kernel`
- **随机数**: `rng_pixel`, `rng_offset`
- **路径标志**: `flag`（PathRayFlag）, `mnee`（PathRayMNEE）
- **体积光学深度**: `optical_depth`
- **MIS 参数**: `mis_ray_pdf`, `mis_ray_object`, `mis_origin_n`, `min_ray_pdf`
- **路径终止**: `continuation_probability`
- **光谱数据**: `throughput`, `unguided_throughput`, `pass_diffuse_weight`, `pass_glossy_weight`, `denoising_feature_throughput`
- **排序键**: `shader_sort_key`

### `ray` 结构（紧凑布局）
光线参数：`P`(起点), `D`(方向), `dP`(位置微分), `dD`(方向微分), `tmin`, `tmax`, `time`, `previous_dt`(光线树用)。

### `isect` 结构（紧凑布局）
场景交点结果：`t`(距离), `u`/`v`(重心坐标), `prim`(图元索引), `object`(对象索引), `type`(图元类型)。

### `subsurface` 结构（紧凑布局）
次表面散射闭包参数：`albedo`(反照率), `radius`(散射半径), `anisotropy`(各向异性), `N`(法线)。仅在 `KERNEL_FEATURE_SUBSURFACE` 启用时存在。

### `volume_stack` 数组结构
体积栈：每个条目包含 `object` 和 `shader`，记录光线当前穿越的体积对象。

### `guiding` 结构
路径引导状态：`path_segment`(OpenPGL路径段指针), `use_surface_guiding`/`use_volume_guiding`(引导启用标志), `surface_guiding_sampling_prob`/`volume_guiding_sampling_prob`(引导采样概率), `sample_surface_guiding_rand`/`sample_volume_guiding_rand`(引导随机数), `bssrdf_sampling_prob`(BSSRDF采样概率)。

### `shadow_link` 结构
阴影链接状态：`dedicated_light_weight`(专用灯光权重), `last_isect_prim`/`last_isect_object`(上一次交点备份)。

## 依赖关系

- **内部头文件**: 无直接 #include（通过宏包含机制使用）
- **被引用**: `state.h`（生成 CPU/GPU 状态结构体）、`state_util.h`（生成状态复制代码）、`src/integrator/path_trace_work_gpu.cpp`（GPU 内存分配）、`kernel/types.h`

## 实现细节 / 关键算法

1. **紧凑打包优化**: `ray`、`isect`、`subsurface` 使用 `KERNEL_STRUCT_BEGIN_PACKED` 声明为紧凑结构。在 GPU 上启用打包布局时，这些结构的所有字段可在单次内存事务中读写，显著减少全局内存访问延迟。

2. **特性条件字段**: 每个字段关联特性标志，如 `KERNEL_FEATURE_SUBSURFACE`、`KERNEL_FEATURE_VOLUME`、`KERNEL_FEATURE_PATH_GUIDING` 等。编译时未启用的特性对应字段将被省略。

3. **整数位宽优化**: 弹射计数器使用 `uint16_t`，标志使用 `uint32_t`，像素索引使用 `uint32_t`。通过延迟乘以 `pass_stride`（在写入时计算），使像素索引能用 32 位整数存储。

4. **路径引导指针**: `guiding.path_segment` 在路径引导启用时为 `openpgl::cpp::PathSegment*` 指针，否则降级为 `uint64_t` 占位符以保持结构体布局一致。

## 关联文件

- `shadow_state_template.h` - 阴影路径状态的对应模板
- `state.h` - 消费模板生成最终数据结构
- `state_util.h` - 消费模板生成状态读写工具
- `path_state.h` - 操作这些状态字段的高层函数
