# subsurface_random_walk.h - 随机游走次表面散射

## 概述

`subsurface_random_walk.h` 实现了基于随机游走的次表面散射模拟，是 Cycles 中最精确的次表面散射方法。该实现基于 Chiang, Kutz & Burley 的 SIGGRAPH 2016 论文 "Practical and Controllable Subsurface Scattering for Production Path Tracing"，结合了 Wrenninge 等人的各向异性相函数扩展，并使用 Dwivedi 零方差采样技术进行引导以加速收敛。

## 核心函数

### `subsurface_random_walk_remap`
将表面反照率(albedo)和散射距离(d)重映射为体积传输系数。通过多项式拟合（7阶关于各向异性参数g的多项式，系数A-F）计算单次散射反照率(alpha)，然后推导消光系数(sigma_t)。

### `subsurface_random_walk_coefficients`
批量处理所有光谱通道的系数转换：
- 调用 `subsurface_random_walk_remap` 计算每通道的 sigma_t 和 alpha
- 从吞吐量中除去反照率（将通过散射重新添加）
- 对低反照率值进行平滑钳制（最小 alpha = 0.2），避免相函数发散

### Dwivedi 采样函数
- **`eval_phase_dwivedi`**: 评估 Dwivedi 相函数 PDF（Meng et al. 2016, Eq. 9）。
- **`sample_phase_dwivedi`**: 采样 Dwivedi 相函数方向（基于 Eq. 10，使用预计算的 `phase_log`）。
- **`diffusion_length_dwivedi`**: 计算 Dwivedi 扩散长度参数（d'Eon & Krivanek 2020, Eq. 67）。

### `direction_from_cosine`
从余弦角和随机方位角构造三维方向向量。

### `subsurface_random_walk_pdf`
计算随机游走的距离采样PDF，根据是否命中表面返回 `T`（透射率）或 `sigma_t * T`。

### `subsurface_random_walk`
随机游走的主函数，执行完整的介质内随机游走：
1. 从积分器状态读取入射信息（位置、方向、法线、散射参数）
2. 转换为体积系数（sigma_t, alpha, sigma_s）
3. 计算 Dwivedi 扩散长度和相函数参数
4. 执行最多 `BSSRDF_MAX_BOUNCES` 次随机游走步骤：
   - 采样颜色通道（MIS平衡启发式）
   - 选择引导或经典采样策略
   - 采样散射方向（Dwivedi引导或 Henyey-Greenstein 相函数）
   - 采样距离并执行局部光线投射
   - 第一次弹射时探测对面界面距离（用于反向引导）
   - 计算MIS权重组合（引导/经典 x 前向/反向 x RGB通道，共9个估计器）
   - 更新吞吐量
5. 若命中表面则返回成功

## 依赖关系

- **内部头文件**:
  - `kernel/bvh/bvh.h` - BVH 局部交点查找
  - `kernel/closure/volume.h` - 体积传输函数
  - `kernel/integrator/guiding.h` - 路径引导记录
  - `kernel/integrator/path_state.h` - 随机数生成
  - `util/color.h` - 颜色工具
- **被引用**: `subsurface.h`（当路径标志为 `PATH_RAY_SUBSURFACE_RANDOM_WALK` 时调用）

## 实现细节 / 关键算法

1. **Dwivedi 零方差引导**: 利用漫反射理论的最优采样方向分布，在法线方向上偏置散射方向采样。引导强度由扩散长度参数 `v` 控制，`v` 越接近1（低反照率）引导越强。前向/反向引导分别针对入射面和对面界面。

2. **引导/经典混合策略**: `guided_fraction` 控制引导采样和经典 HG 相函数采样的混合比例，默认为 `1 - max(0.5, |g|^0.125)`。高各向异性时经典采样更重要，低各向异性时引导采样更有效。

3. **对面界面探测**: 第一次弹射时使用延长的光线（最多10倍平均自由程）探测对面界面距离。发现对面后启用反向 Dwivedi 采样，概率基于 sigmoid 函数 `1/(1 + exp((d - 2x)/v))`（取决于游走点到对面的相对距离）。

4. **相似性简化**: 当弹射次数超过 `SUBSURFACE_RANDOM_WALK_SIMILARITY_LEVEL`（默认9）时，将各向异性相函数退化为各向同性（`g=0`），使用等效的缩减消光系数 `sigma_t* = sigma_t - sigma_s + sigma_s*(1-g)`，减少深层游走的计算复杂度。

5. **多级MIS**: 最终 PDF 组合了三个层级的 MIS：(1) 引导 vs 经典采样，(2) 前向 vs 反向引导，(3) RGB 颜色通道。共计最多9个不同的估计器通过平衡启发式组合。

## 关联文件

- `subsurface.h` - 次表面散射调度入口
- `subsurface_disk.h` - 圆盘采样方法（替代方案，更快但精度较低）
- `kernel/closure/bssrdf.h` - BSSRDF 闭包数学基础
- `kernel/closure/volume.h` - 体积传输函数（透射率计算）
