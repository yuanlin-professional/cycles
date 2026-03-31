# shade_light.h - 光源着色

## 概述

本文件实现了光源着色内核，处理路径追踪过程中射线直接命中光源（如面光源、点光源、聚光灯等）时的着色逻辑。该内核负责从交点构建光源样本、评估光源着色器、计算 MIS 权重，并将贡献写入渲染缓冲区。光源命中后路径继续前进（光源被视为透明表面），直到达到透明弹射上限。

## 核心函数

### integrate_light()

- **签名**: `ccl_device_inline void integrate_light(KernelGlobals kg, IntegratorState state, ccl_global float *ccl_restrict render_buffer)`
- **功能**: 评估光源交点处的光照贡献。执行流程：
  1. 从积分器状态读取交点数据
  2. 记录路径引导的光源表面段
  3. 读取射线属性（位置、方向、时间）和路径标志
  4. 将射线 `tmin` 推进到光源交点之后（避免重复命中）
  5. 调用 `light_sample_from_intersection` 从交点构建光源样本
  6. 检查光源着色器对当前路径的可见性
  7. 评估光源着色器（支持节点着色器）
  8. 计算前向 MIS 权重（`light_sample_mis_weight_forward_lamp`）
  9. 通过路径引导记录表面发射
  10. 写入渲染缓冲区（`film_write_surface_emission`）

### integrator_shade_light()

- **签名**: `ccl_device void integrator_shade_light(KernelGlobals kg, IntegratorState state, ccl_global float *ccl_restrict render_buffer)`
- **功能**: 光源着色内核的主入口。调用 `integrate_light` 执行光照评估后，递增透明弹射计数（将光源解释为透明表面）。若透明弹射达到上限，终止路径；否则调度 `INTERSECT_CLOSEST` 内核继续射线前进，以处理光源之后可能存在的几何体。

## 依赖关系

- **内部头文件**:
  - `kernel/film/light_passes.h` - 光线通道写入（`film_write_surface_emission`）
  - `kernel/light/light.h` - 光源工具函数
  - `kernel/light/sample.h` - 光源采样、着色器评估和 MIS 权重
- **被引用**:
  - `kernel/integrator/megakernel.h` - 超级内核循环
  - `kernel/device/optix/kernel_osl.cu` - OptiX OSL 内核
  - `kernel/device/gpu/kernel.h` - GPU 内核

## 实现细节 / 关键算法

1. **光源作为透明表面**: 光源被视为透明表面计入透明弹射。这允许射线穿过光源继续追踪后方的几何体，但也存在精度问题（TODO 注释提到可能因重复命中同一光源而陷入无限循环）。
2. **MIS 一致性**: 使用前向 MIS 权重（`light_sample_mis_weight_forward_lamp`），与直接光照采样中的反向 MIS 权重配对，确保多重重要性采样的无偏性。
3. **射线推进**: 通过 `intersection_t_offset(isect.t)` 将射线的 `tmin` 推进到交点稍后的位置，以避免重复命中同一光源。
4. **法线用于 MIS**: 使用 `mis_origin_n`（来自上一次表面弹射的法线）参与光源 MIS 权重计算，确保与 `integrate_surface_direct_light` 中的采样概率一致。
5. **路径引导**: 在光源着色开始时记录 `guiding_record_light_surface_segment`，在评估后记录 `guiding_record_surface_emission`，为引导训练提供完整的光源交互信息。

## 关联文件

- `kernel/integrator/intersect_closest.h` - 命中 `PRIMITIVE_LAMP` 时调度到本内核
- `kernel/integrator/shade_background.h` - 处理远距光和背景光（对比参考）
- `kernel/integrator/shade_dedicated_light.h` - 阴影链接专用光源着色（对比参考）
- `kernel/light/sample.h` - 光源着色器评估和 MIS 权重
