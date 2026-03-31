# bevel.h - 倒角着色器节点的SVM实现

## 概述

`bevel.h` 实现了着色器虚拟机(SVM)中的倒角(Bevel)节点。该节点通过光线追踪采样附近表面的法线并进行加权平均，在着色阶段模拟几何倒角效果，无需实际修改网格几何。该功能需要 `__SHADER_RAYTRACE__` 编译选项支持，且使用了 BSSRDF 重要性采样策略来高效采样邻近表面。

## 核心函数

### `svm_bevel_cubic_eval` / `svm_bevel_cubic_pdf`
- **功能**: 平面立方 BSSRDF 衰减函数，计算 `10*(R-r)^3 / (pi*R^5)` 形式的归一化衰减。`svm_bevel_cubic_pdf` 直接等价于 `svm_bevel_cubic_eval`，用于概率密度评估。

### `svm_bevel_cubic_quintic_root_find`
- **签名**: `ccl_device_forceinline float svm_bevel_cubic_quintic_root_find(const float xi)`
- **功能**: 使用 Newton-Raphson 迭代求解五次方程 `10x^2 - 20x^3 + 15x^4 - 4x^5 - xi = 0`。该方程是立方衰减核在圆盘上积分的 CDF 逆函数，用于重要性采样。
- **收敛性**: 通常在 2-4 次迭代内收敛，极端区间(0.02/0.98外)可能需要 10 次。

### `svm_bevel_cubic_sample`
- **签名**: `ccl_device void svm_bevel_cubic_sample(const float radius, const float xi, float *r, float *h)`
- **功能**: 采样圆盘上的一个点，返回径向距离 `r` 和圆盘高度 `h`（从 `h^2 + r^2 = Rm^2` 计算）。

### `svm_bevel` / `__direct_callable__svm_node_bevel`（OptiX 版本）
- **签名**: `ccl_device float3 svm_bevel(KernelGlobals kg, ConstIntegratorState state, ShaderData *sd, const float radius, const int num_samples)`
- **功能**: 核心倒角法线计算函数。返回经过邻近表面法线平均后的新着色法线。
- **算法**:
  1. 提前退出：半径 <= 0、采样数 < 1、无对象、BVH 未构建、模糊间接光线(`min_ray_pdf < 8.0`)
  2. 初始化 LCG 随机数状态
  3. 对每个采样：
     - 随机选择三个局部坐标轴之一（Ng 概率 50%，T/B 各 25%）
     - 在选定平面上用立方 BSSRDF 衰减核采样圆盘上的点
     - 构建探测光线：从偏移点沿选定轴负方向射出
     - 调用 `scene_intersect_local` 在同一对象上做多重相交（最多 `LOCAL_MAX_HITS` 次命中）
     - 对每个命中点：获取位置、几何法线、平滑法线，执行变换
     - 使用多重重要性采样(MIS)的功率启发式计算权重
     - 累加 `weight * N`
  4. 归一化累加法线，如果为零则回退到原始法线

### `svm_node_bevel`
- **签名**: `template<uint node_feature_mask, typename ConstIntegratorGenericState> void svm_node_bevel(...)`
- **功能**: SVM 节点入口。解包参数，调用 `svm_bevel`，并可选地将倒角法线与输入法线混合。

## 依赖关系

- **内部头文件**:
  - `kernel/bvh/bvh.h` — BVH 光线相交（局部相交）
  - `kernel/geom/motion_triangle.h` — 运动模糊三角形
  - `kernel/geom/triangle.h` — 三角形顶点与法线
  - `kernel/geom/triangle_intersect.h` — 三角形相交测试
  - `kernel/integrator/path_state.h` — 路径状态与随机数
  - `kernel/svm/util.h` — SVM 栈操作工具
- **被引用**: `kernel/osl/services.cpp`

## 实现细节 / 关键算法

1. **BSSRDF 重要性采样策略**: 采样策略来源于 SIGGRAPH 2013 论文 "BSSRDF Importance Sampling"。使用三轴平面投影采样（类似于 BSSRDF 的采样策略），在法线方向(Ng)上分配 50% 的概率，切线(T)和副切线(B)方向各 25%。

2. **立方衰减核**: 使用 `(Rm - r)^3` 形式的衰减函数（与平面立方 BSSRDF 相同），控制邻近表面对最终法线的贡献权重。距离越近的表面贡献越大。

3. **多重重要性采样(MIS)**: 对三个采样轴使用功率启发式(power heuristic)进行 MIS 加权：`w = pdf_N / (pdf_N^2 + pdf_T^2 + pdf_B^2)`。当实际命中数超过 `LOCAL_MAX_HITS` 时，额外乘以补偿系数 `num_hits / LOCAL_MAX_HITS`。

4. **模糊光线优化**: 当 `min_ray_pdf < 8.0`（间接模糊光线）时跳过倒角计算，避免在不可见细节上浪费计算资源。

5. **法线混合**: 如果用户提供了自定义法线输入 `normal_offset`，最终法线通过 `normalize(ref_N + (bevel_N - sd->N))` 将倒角效果叠加到输入法线上。

## 关联文件

- `kernel/svm/ao.h` — 同样使用光线追踪的 AO 节点
- `kernel/bvh/bvh.h` — BVH 加速结构
- `kernel/osl/services.cpp` — OSL 版本的倒角实现
