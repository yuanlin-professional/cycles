# subsurface_disk.h - 基于圆盘的次表面散射采样

## 概述

`subsurface_disk.h` 实现了基于圆盘投影的次表面散射采样算法，对应 Burley 散射剖面。该方法源自 SIGGRAPH 2013 论文 "BSSRDF Importance Sampling"（ImageWorks），通过在表面法线方向上投影圆盘，发射局部光线查找同一对象上的附近出口点，然后使用重要性重采样从多个候选命中点中选择最终出口。

## 核心函数

### `subsurface_disk_eval`
评估圆盘采样的次表面散射剖面权重。给定散射半径、圆盘采样半径(`disk_r`)和实际距离(`r`)，计算 BSSRDF 评估值除以采样PDF的比值。

### `subsurface_disk`
圆盘采样的主函数，执行完整的次表面散射步骤：
1. 从积分器状态读取着色点信息（位置P、法线Ng、对象、散射半径等）
2. 在局部坐标系中随机选择投影轴（50% N轴、25% T轴、25% B轴）
3. 在选定平面上采样圆盘点，使用 BSSRDF 分布确定采样半径和高度
4. 构造从圆盘上方穿过到下方的局部光线
5. 调用 `scene_intersect_local` 查找同一对象上的交点（最多 `BSSRDF_MAX_HITS` 个）
6. 对每个命中点评估散射剖面，使用三轴MIS的幂启发式合并权重
7. 通过重要性重采样（按权重概率）从候选点中选择最终出口点
8. 更新路径吞吐量和光线状态

## 依赖关系

- **内部头文件**:
  - `kernel/bvh/bvh.h` - BVH 局部交点查找
  - `kernel/closure/bssrdf.h` - BSSRDF 采样和评估函数
  - `kernel/geom/object.h` - 对象变换
  - `kernel/integrator/guiding.h` - 路径引导
  - `kernel/integrator/path_state.h` - 随机数生成
  - `kernel/util/differential.h` - 微分工具
- **被引用**: `subsurface.h`（当路径标志为 `PATH_RAY_SUBSURFACE_DISK` 时调用）

## 实现细节 / 关键算法

1. **三轴投影MIS**: 在法线(N)、切线(T)和副切线(B)三个方向上进行圆盘投影，通过多重重要性采样合并三个方向的采样概率。选择概率为 N:T:B = 50%:25%:25%，MIS使用幂启发式（`power heuristic`），比平衡启发式效果略好。

2. **BSSRDF分布采样**: 使用 `bssrdf_sample` 函数根据散射半径的每通道分布采样圆盘半径和高度，确保采样分布与实际散射剖面匹配。

3. **局部交点查找**: `scene_intersect_local` 仅查找与指定对象的交点，使用BVH的局部遍历模式。当交点数超过 `BSSRDF_MAX_HITS` 时，使用LCG随机数进行储层采样选择随机子集。

4. **重要性重采样**: 从多个候选命中点中，按每个点的权重均值（`average(fabs(weight))`）进行概率重采样，选择单个出口点。重采样的补偿因子 `sum_weights / sample_weight` 确保估计器无偏。

5. **背面处理**: 当入射光线从背面进入时（`PATH_RAY_SUBSURFACE_BACKFACING`），翻转命中点的几何法线以确保正确的散射剖面评估。

## 关联文件

- `subsurface.h` - 次表面散射调度入口
- `subsurface_random_walk.h` - 随机游走方法（替代方案）
- `kernel/closure/bssrdf.h` - BSSRDF 散射剖面数学
