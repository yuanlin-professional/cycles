# shadow_linking.h - 阴影链接调度与状态管理

## 概述

`shadow_linking.h` 实现了阴影链接(Shadow Linking)功能的调度逻辑。阴影链接允许用户精确控制哪些对象向哪些灯光投射阴影，实现选择性的光影关系。该文件提供了场景级别的阴影链接需求检测、交点数据的保存/恢复，以及专用阴影光线内核的调度接口。

## 核心函数

### `shadow_linking_scene_need_shadow_ray`
检查当前场景配置是否需要额外的阴影链接光线。若内核特性标志中未包含 `KERNEL_FEATURE_SHADOW_LINKING`，则不需要额外光线。

### `shadow_linking_store_last_primitives`
将主路径的当前交点信息（`prim` 和 `object`）保存到 `shadow_link` 状态中。因为阴影链接复用了主路径的交点存储来传递灯光信息，需要在覆盖前保存原始数据。

### `shadow_linking_restore_last_primitives`
从 `shadow_link` 状态中恢复先前保存的交点信息，使 `intersect_closest` 内核在表面弹射后能正确使用原始交点数据。

### `shadow_linking_schedule_intersection_kernel<current_kernel>`
模板函数，在条件满足时调度 `DEVICE_KERNEL_INTEGRATOR_INTERSECT_DEDICATED_LIGHT` 内核。返回 `true` 表示已调度阴影链接内核（调用方应立即返回），`false` 表示不需要阴影链接处理。

## 依赖关系

- **内部头文件**: `kernel/integrator/state_flow.h` - 路径控制流与内核调度
- **被引用**: `shade_surface.h`（表面着色后调度）、`shade_volume.h`（体积散射后调度）、`intersect_dedicated_light.h`（专用灯光交点内核）

## 实现细节 / 关键算法

1. **交点数据复用**: 阴影链接在主路径交点结构体和着色内核之间传递灯光信息，而非使用独立存储。`store`/`restore` 函数对确保数据不会被覆盖丢失。

2. **调度条件**: 阴影链接光线仅在以下条件下被调度：
   - 场景包含阴影链接特性
   - 当前弹射不是透明弹射（透明弹射的灯光通过主路径累积）
   - 当前弹射不是 BSSRDF 弹射（继续进入次表面散射内核）

3. **远光灯处理**: 注释指出远光灯可能使用阴影链接但不计入 `use_light_mis`，因此条件检查较为保守，存在进一步优化空间。

## 关联文件

- `shade_surface.h` - 表面着色后调度阴影链接
- `shade_volume.h` - 体积散射后调度阴影链接
- `intersect_dedicated_light.h` - 专用灯光交点内核实现
- `state_template.h` - 定义 `shadow_link` 状态存储结构
