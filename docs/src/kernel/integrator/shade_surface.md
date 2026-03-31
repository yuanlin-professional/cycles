# shade_surface.h - 表面着色积分内核

## 概述

`shade_surface.h` 是 Cycles 路径追踪器中表面着色的核心文件，负责光线击中表面后的完整着色流程。该文件实现了直接光照采样、双向散射分布函数(BSDF)/次表面散射(BSSRDF)弹射、自发光计算、环境光遮蔽(AO)以及 holdout 处理等功能。它是积分器中最关键的着色内核之一，将光线-表面交互转化为最终的颜色贡献。

## 核心函数

### `integrate_surface_shader_setup`
从积分器状态中读取交点和光线数据，初始化 `ShaderData` 结构体，为后续着色评估做准备。

### `integrate_surface_ray_offset`
处理三角形图元的光线偏移，通过自交测试判断是否需要对光线起点进行偏移，避免因浮点精度问题导致邻近三角形被误命中。

### `integrate_surface_holdout`
评估 holdout（镂空）透明度并写入渲染缓冲区。若表面完全为 holdout 则终止路径。

### `integrate_surface_emission`
评估表面自发光闭包，结合多重重要性采样(MIS)权重将发光贡献写入胶片缓冲区。支持光照链接和阴影链接的过滤。

### `integrate_surface_ray_portal`
处理光线门户(Ray Portal)闭包，将光线传送到门户目标位置并更新路径状态。

### `integrate_direct_light_shadow_init_common`
初始化阴影路径的公共部分，包括分支阴影内核、复制体积栈、写入阴影光线以及从主路径复制状态到阴影路径。

### `integrate_surface_direct_light<node_feature_mask>`
模板函数，执行直接光照的下一事件估计(NEE)。采样光源位置、评估光源着色器、计算 BSDF 响应、应用 MIS 权重，并创建阴影光线。支持 MNEE（流形下一事件估计）焦散采样。

### `integrate_surface_bsdf_bssrdf_bounce`
采样 BSDF 或 BSSRDF 闭包以确定弹射方向。支持路径引导(Path Guiding)采样策略，更新路径吞吐量和 MIS 状态。如果采样到 BSSRDF，调度次表面散射内核。

### `integrate_surface_volume_only_bounce`
仅用于体积边界表面的透射弹射，不做实际的表面着色，仅更新光线起始距离。

### `integrate_surface_terminate`
基于继续概率决定是否终止路径。若路径标记为在下一个表面终止，则直接返回终止。

### `integrate_surface_ao`
计算环境光遮蔽通道，在表面法线的余弦加权半球上采样方向，创建短距 AO 阴影光线。

### `integrate_surface<node_feature_mask>`
主表面积分函数，协调完整的表面着色流程：设置着色器数据 -> 评估着色器 -> 处理次表面散射出口 -> 过滤闭包 -> holdout -> 自发光 -> 路径终止 -> 渲染通道写入 -> 直接光照 -> AO -> BSDF 弹射。

### `integrator_shade_surface`
最外层入口函数模板，调用 `integrate_surface` 并根据结果调度下一个内核（终止、阴影链接或继续追踪）。

### `integrator_shade_surface_raytrace`
带光线追踪节点支持的着色变体。

### `integrator_shade_surface_mnee`
带 MNEE 焦散支持的着色变体。

## 依赖关系

- **内部头文件**:
  - `kernel/integrator/path_state.h` - 路径状态管理
  - `kernel/integrator/surface_shader.h` - 表面着色器评估
  - `kernel/integrator/mnee.h` - 流形下一事件估计
  - `kernel/integrator/guiding.h` - 路径引导
  - `kernel/integrator/shadow_linking.h` - 阴影链接
  - `kernel/integrator/subsurface.h` - 次表面散射
  - `kernel/integrator/volume_stack.h` - 体积栈
  - `kernel/film/data_passes.h`, `denoising_passes.h`, `light_passes.h` - 胶片通道
  - `kernel/light/sample.h` - 光源采样
  - `kernel/geom/triangle.h`, `motion_triangle.h` - 几何体
- **被引用**: `megakernel.h`, `intersect_dedicated_light.h`, `shade_dedicated_light.h`, GPU/OptiX 内核调度文件

## 实现细节 / 关键算法

1. **光线偏移策略**: 对于三角形图元，先执行自交测试（`ray_triangle_intersect_self`），只有在测试失败时才沿几何法线偏移光线起点，减少不必要的偏移造成的视觉瑕疵。

2. **直接光照 MIS**: 直接光照使用光源采样PDF和BSDF采样PDF的多重重要性采样权重 `light_sample_mis_weight_nee`，平衡两种采样策略的贡献。

3. **MNEE 焦散**: 当光源标记为使用焦散且当前表面为焦散接收器时，调用 `kernel_path_mnee_sample` 进行流形游走，生成穿过光滑折射界面的阴影光线。

4. **路径引导集成**: 在 BSDF 采样中，当路径引导启用时，使用引导分布与原始 BSDF 的混合采样，并通过 `unguided_throughput` 维护未引导吞吐量以支持俄罗斯轮盘终止。

5. **内核调度**: 着色完成后根据路径标签决定下一个内核：次表面散射路径进入 `INTERSECT_SUBSURFACE`，普通路径进入 `INTERSECT_CLOSEST`，阴影链接路径进入 `INTERSECT_DEDICATED_LIGHT`。

## 关联文件

- `surface_shader.h` - 提供 BSDF 评估和采样的底层实现
- `shade_volume.h` - 体积着色的对应实现
- `shadow_linking.h` - 阴影链接调度逻辑
- `subsurface.h` - 次表面散射弹射入口
- `intersect_closest.h` - 最近交点查找内核
