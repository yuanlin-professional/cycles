# integrator.h / integrator.cpp - 路径追踪积分器参数配置

## 概述

本文件定义了 Cycles 渲染器的路径追踪积分器配置。`Integrator` 类继承自 `Node`，包含控制路径追踪行为的所有参数，包括光线反弹深度、采样模式、体积光线步进、路径引导、焦散控制、降噪器配置、自适应采样等。该类负责将这些参数打包传输到设备端的 `KernelIntegrator` 结构体，并生成 Sobol 采样序列查找表。

## 类与结构体

### Integrator

- **继承**: `Node`（节点系统基类）
- **功能**: 管理路径追踪积分器的全部参数，是渲染质量和性能平衡的核心控制点。

#### 反弹深度参数
- `min_bounce` / `max_bounce` — 全局最小/最大反弹次数
- `max_diffuse_bounce` — 漫射最大反弹
- `max_glossy_bounce` — 光泽最大反弹
- `max_transmission_bounce` — 透射最大反弹
- `max_volume_bounce` — 体积最大反弹
- `transparent_min_bounce` / `transparent_max_bounce` — 透明材质反弹范围

#### 环境遮蔽（AO）参数
- `ao_bounces` — AO 近似的反弹次数
- `ao_factor` — AO 强度因子
- `ao_distance` — AO 最大距离
- `ao_additive_factor` — AO 叠加因子

#### 体积参数
- `volume_ray_marching` — 是否使用有偏的光线步进
- `volume_max_steps` — 体积最大步数
- `volume_step_rate` — 体积步进速率

#### 路径引导参数
- `use_guiding` — 是否启用路径引导
- `deterministic_guiding` — 是否使用确定性引导
- `use_surface_guiding` / `surface_guiding_probability` — 表面引导开关及概率
- `use_volume_guiding` / `volume_guiding_probability` — 体积引导开关及概率
- `guiding_training_samples` — 引导训练样本数
- `use_guiding_direct_light` — 是否对直接光照使用引导
- `use_guiding_mis_weights` — 是否使用 MIS 权重
- `guiding_distribution_type` — 引导分布类型（Parallax-Aware VMM / 方向四叉树 / VMM）
- `guiding_directional_sampling_type` — 方向采样类型（MIS / RIS / 粗糙度）
- `guiding_roughness_threshold` — 引导粗糙度阈值

#### 焦散与过滤
- `caustics_reflective` / `caustics_refractive` — 是否计算反射/折射焦散
- `filter_glossy` — 光泽过滤强度
- `use_direct_light` / `use_indirect_light` / `use_diffuse` / `use_glossy` / `use_transmission` / `use_emission` — 光照分量开关

#### 采样参数
- `seed` — 随机种子
- `sample_clamp_direct` / `sample_clamp_indirect` — 直接/间接采样钳制值
- `motion_blur` — 运动模糊开关
- `aa_samples` — 抗锯齿采样数（`MAX_SAMPLES = 2^24`）
- `use_sample_subset` / `sample_subset_offset` / `sample_subset_length` — 样本子集采样（用于分块渲染）
- `sampling_pattern` — 采样模式（Sobol-Burley / 预制表 Sobol / 蓝噪声纯 / 蓝噪声舍入 / 蓝噪声首位）
- `scrambling_distance` — 扰动距离

#### 光源采样
- `use_light_tree` — 是否使用光源树优化多光源采样
- `light_sampling_threshold` — 光源采样阈值（低于此值的光源使用俄罗斯轮盘赌）

#### 自适应采样
- `use_adaptive_sampling` — 是否使用自适应采样
- `adaptive_min_samples` — 自适应最小样本数
- `adaptive_threshold` — 自适应噪声阈值

#### 降噪器参数
- `use_denoise` — 是否启用降噪
- `denoiser_type` — 降噪器类型（OptiX / OpenImageDenoise）
- `denoise_start_sample` — 开始降噪的样本数
- `use_denoise_pass_albedo` / `use_denoise_pass_normal` — 是否使用反照率/法线辅助通道
- `denoiser_prefilter` — 预滤波模式（无 / 快速 / 精确）
- `denoiser_quality` — 降噪质量（高 / 平衡 / 快速）
- `denoise_use_gpu` — 是否在 GPU 上降噪

#### 更新标志
| 标志 | 说明 |
|------|------|
| `AO_PASS_MODIFIED` | AO 通道已修改 |
| `OBJECT_MANAGER` | 对象管理器更新 |

- **关键方法**:
  - `device_update(Device*, DeviceScene*, Scene*)` — 将所有参数同步到设备内核，含 Sobol 序列生成
  - `device_free(Device*, DeviceScene*, bool)` — 释放设备端采样模式查找表
  - `tag_update(Scene*, uint32_t)` — 按标志位触发更新
  - `get_kernel_features()` — 返回所需内核特性标志（AO 叠加、光源树）
  - `get_adaptive_sampling()` — 获取自适应采样配置
  - `get_denoise_params()` — 获取降噪参数
  - `get_guiding_params(Device*)` — 获取路径引导参数（需设备支持检查）

## 核心函数

所有逻辑封装在 `Integrator` 类方法中。

## 依赖关系

- **内部头文件**:
  - `kernel/types.h` — `KernelIntegrator` 等内核类型
  - `device/denoise.h` — 降噪参数和类型枚举
  - `graph/node.h` — 节点系统基类
  - `integrator/adaptive_sampling.h` — `AdaptiveSampling` 结构体
  - `integrator/guiding.h` — `GuidingParams` 结构体
  - `scene/background.h`、`scene/bake.h`、`scene/camera.h`、`scene/film.h`、`scene/light.h`、`scene/object.h`、`scene/scene.h`、`scene/shader.h`、`scene/tabulated_sobol.h`（cpp 中引用）
- **被引用**: `scene/light.cpp`、`scene/film.cpp`、`scene/background.cpp`、`scene/scene.cpp`、`scene/shader.cpp`、`scene/object.cpp`、`scene/volume.cpp`、`scene/bake.cpp`、`session/session.cpp`、`session/tile.cpp`、`integrator/render_scheduler.cpp`、`hydra/render_delegate.cpp`、`app/` 等约 17 个文件

## 实现细节 / 关键算法

1. **反弹深度偏移**: 内核中的反弹计数从 1 开始，因此所有反弹参数在传输到内核时加 1。`max_bounce = 0` 表示仅直接光照，无全局光照。透明反弹例外：0 表示场景中无透明反弹。

2. **透明阴影检测**: 遍历所有着色器，检查是否存在具有透明阴影或体积的着色器。只有在场景中确实需要时才启用 `transparent_shadows`，以提升无透明材质场景的性能。

3. **闭包过滤系统**: 通过 `filter_closures` 位掩码控制哪些 BSDF 分量被评估。烘焙模式额外过滤透明闭包以确保只烘焙目标对象本身。关闭间接光照时将最大反弹设为 1。

4. **采样模式处理**:
   - `SAMPLING_PATTERN_TABULATED_SOBOL`: 预生成四维 Sobol 序列表，大小为 `sequence_size * NUM_TAB_SOBOL_PATTERNS * NUM_TAB_SOBOL_DIMENSIONS`，使用 `TaskPool` 并行生成
   - `SAMPLING_PATTERN_BLUE_NOISE_ROUND`: 将序列长度向上取整为 2 的幂，然后转换为纯蓝噪声模式
   - `SAMPLING_PATTERN_BLUE_NOISE_FIRST`: 序列长度减 1
   - 蓝噪声模式对种子进行哈希处理以确保正确的扰动

5. **自适应采样参数计算** (`get_adaptive_sampling`):
   - 阈值为 0 时自动设为 `max(0.001, 1/aa_samples)`
   - 最小样本数为 0 时自动计算：`ceil(16 / threshold^0.3)`，范围 [4, ...]
   - 阈值最终乘以 5 的经验因子
   - 使用样本子集时，阈值按 `sqrt(subset/total)` 比例缩放
   - 自适应步长固定为 16（必须为 2 的幂以支持位运算）

6. **路径引导参数**: 引导功能需要设备支持（`device->info.has_guiding`）。粗糙度阈值平方化以匹配 Blender/Cycles 的线性行为约定。

7. **光源采样阈值**: 当使用光源采样阈值（非光源树模式）时，计算逆阈值 = `exposure / threshold`，用于低贡献光源的俄罗斯轮盘赌终止。

## 关联文件

- `scene/film.h` / `scene/film.cpp` — 自适应采样和降噪影响通道配置
- `scene/light.h` / `scene/light.cpp` — 光源树开关
- `scene/camera.h` — 运动模糊联动
- `scene/tabulated_sobol.h` — 预制表 Sobol 序列生成
- `integrator/adaptive_sampling.h` — `AdaptiveSampling` 数据结构
- `integrator/guiding.h` — `GuidingParams` 数据结构
- `device/denoise.h` — `DenoiseParams` 数据结构
- `kernel/types.h` — `KernelIntegrator` 设备端数据结构
