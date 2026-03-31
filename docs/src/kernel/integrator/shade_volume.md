# shade_volume.h - 体积着色与传输积分内核

## 概述

`shade_volume.h` 是 Cycles 路径追踪器中体积渲染的核心实现文件，负责光线在参与介质（体积）中的传播、散射和发光计算。该文件实现了基于八叉树的分层DDA光线行进、无偏透射率估计（基于伸缩求和法）、等角采样、距离采样以及体积散射的多重重要性采样。它是整个体积渲染管线中最复杂的文件之一。

## 核心函数

### 体积着色器评估
- **`volume_shader_eval_extinction`**: 在给定点评估消光系数（sigma_t），仅计算透明消光而不存储相函数。
- **`volume_shader_sample`**: 评估着色器获取完整的体积系数：吸收(sigma_a)、散射(sigma_s)和发光(emission)。

### 八叉树光线行进
- **`OctreeTracing`**: 结构体，封装八叉树遍历状态，包括当前叶节点、活动光线区间、八叉树空间坐标等。实现了坐标空间变换 `to_octree_space`、体素相交 `ray_voxel_intersect`、位置推进 `find_next_pos` 等方法。
- **`volume_voxel_get`**: 从当前位置查找对应的八叉树叶节点。
- **`volume_estimate_extrema`**: 当着色器包含光路节点时，在光线上随机采样若干点估算密度极值。
- **`volume_object_get_extrema`**: 获取当前八叉树节点的密度极值，优先使用预烘焙数据。
- **`volume_find_octree_root`**: 根据体积栈条目查找对应的八叉树根节点。
- **`volume_octree_setup`**: 设置当前活动光线段，处理多个重叠八叉树的情况。
- **`volume_octree_advance`**: 推进到下一个相邻叶节点并更新活动区间。
- **`volume_octree_advance_shadow`**: 阴影光线的八叉树推进，合并区间直到光学深度足够大。

### 透射率计算
- **`volume_transmittance`**: 使用伸缩求和法（Misso et al.）计算光线段的无偏透射率，支持几何分布采样去偏项阶数。
- **`volume_shadow_null_scattering`**: 计算阴影光线的体积透射率衰减。

### 采样方法
- **`volume_equiangular_sample` / `volume_equiangular_pdf`**: 等角采样及其概率密度，用于在光源附近高效采样散射位置。
- **`volume_valid_direct_ray_segment`**: 计算光线中直接可见光源的有效段。
- **`volume_emission_integrate`**: 积分体积发光贡献。

### 散射与积分
- **`volume_integrate_step_scattering`**: 在单步行进中执行散射和发光积分。
- **`volume_equiangular_direct_scatter`**: 等角采样的直接散射处理。
- **`volume_direct_scatter_mis`**: 距离采样和等角采样之间的MIS。
- **`volume_sample_indirect_scatter`**: 间接散射采样。
- **`volume_scatter_probability`**: 计算散射概率。
- **`volume_integrate_should_stop`**: 判断体积积分是否应停止。

### 入口函数
- **`integrator_shade_volume_setup`**: 设置体积着色的着色器数据。
- **`integrator_shade_volume`**: 均匀体积着色内核入口。
- **`integrator_shade_volume_ray_marching`**: 非均匀体积的光线行进着色内核入口。

## 依赖关系

- **内部头文件**:
  - `kernel/closure/volume.h` - 体积闭包（相函数）
  - `kernel/integrator/guiding.h` - 路径引导
  - `kernel/integrator/intersect_closest.h` - 最近交点
  - `kernel/integrator/path_state.h` - 路径状态
  - `kernel/integrator/shadow_linking.h` - 阴影链接
  - `kernel/integrator/volume_shader.h` - 体积着色器评估
  - `kernel/integrator/volume_stack.h` - 体积栈
  - `kernel/light/light.h`, `sample.h` - 光源采样
  - `kernel/film/denoising_passes.h`, `light_passes.h` - 胶片通道
  - `kernel/sample/lcg.h` - 线性同余随机数
- **被引用**: `megakernel.h`, `shade_shadow.h`, GPU/OptiX 内核调度文件

## 实现细节 / 关键算法

1. **分层DDA八叉树遍历**: 参考 Laine & Karras 的稀疏体素八叉树方法。将光线段变换到八叉树空间 [1, 2)，通过翻转光线方向使其始终为负方向，简化八分体查找。使用父节点索引代替显式栈以优化GPU性能。

2. **无偏透射率估计（伸缩求和法）**: 基于 Misso et al. 的方法，使用 T_k（有偏估计）加去偏项 (T_{j+1} - T_j)/pmf 的组合，其中阶数 k 根据光学厚度确定，去偏项阶数服从几何分布采样。以2的幂次复用样本。

3. **等角采样**: 实现 Kulla & Fajardo 的等角采样技术，相对于光源位置进行采样，在光源附近区域比距离采样更高效。

4. **密度极值管理**: 每个八叉树叶节点存储预计算的密度极值（最大值和最小值）。对于包含 Light Path 节点的着色器，在运行时采样估算极值，并使用 1.5 倍安全系数。多个重叠体积的极值会被累加。

5. **MIS组合**: 体积积分支持距离采样和等角采样的多重重要性采样组合，根据着色器标志自动选择采样策略。

## 关联文件

- `volume_shader.h` - 体积着色器评估与相函数采样
- `volume_stack.h` - 体积栈的进出管理
- `shade_surface.h` - 表面着色的对应实现
- `shade_shadow.h` - 阴影光线处理
