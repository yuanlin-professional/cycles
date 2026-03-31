# mnee.h - 流形下一事件估计(MNEE)

## 概述

本文件实现了流形下一事件估计（Manifold Next Event Estimation, MNEE）算法，这是一种通过折射表面进行直接光照采样的高级技术。MNEE 通过在"镜面流形"上行走，找到满足费马原理的折射路径，从而高效采样穿过玻璃等折射材质的焦散光线。该算法基于伪牛顿迭代求解器，利用广义半向量约束和流形探索框架实现。

## 核心函数

### mnee_setup_manifold_vertex()

- **签名**: `ccl_device_forceinline void mnee_setup_manifold_vertex(KernelGlobals kg, ccl_private ManifoldVertex *vtx, ccl_private ShaderClosure *bsdf, const float eta, const float2 n_offset, const ccl_private Ray *ray, const ccl_private Intersection *isect, ccl_private ShaderData *sd_vtx)`
- **功能**: 从射线和交点数据设置流形顶点。计算位置、法线、切线空间（位置偏导数 dp/du、dp/dv）、法线偏导数（dn/du、dn/dv），以及几何和闭包信息。对切线空间进行正交化处理，并建立与几何法线一致的参考坐标系。

### mnee_compute_constraint_derivatives()

- **签名**: `bool mnee_compute_constraint_derivatives(const int vertex_count, ccl_private ManifoldVertex *vertices, const ccl_private float3 &surface_sample_pos, const bool light_fixed_direction, const float3 light_sample)`
- **功能**: 计算约束函数及其对所有流形顶点的导数矩阵。对每个顶点计算广义半向量在局部切平面上的投影（约束向量），以及约束关于前一顶点（矩阵 a）、当前顶点（矩阵 b）和后一顶点（矩阵 c）的偏导数。这些构成块三对角矩阵系统。

### mnee_solve_matrix_h_to_x()

- **签名**: `ccl_device_forceinline bool mnee_solve_matrix_h_to_x(const int vertex_count, ccl_private ManifoldVertex *vertices, ccl_private float2 *dx)`
- **功能**: 求解块三对角线性方程组 `A * dx = h`，将约束空间的偏差映射回顶点位移。使用块三对角 LU 分解和回代法。

### mnee_newton_solver()

- **签名**: `ccl_device_forceinline bool mnee_newton_solver(KernelGlobals kg, const ccl_private ShaderData *sd, ccl_private ShaderData *sd_vtx, const ccl_private LightSample *ls, const bool light_fixed_direction, const int vertex_count, ccl_private ManifoldVertex *vertices)`
- **功能**: 牛顿迭代求解器，在镜面流形上行走以找到满足斯涅尔定律的顶点配置。每次迭代：
  1. 计算约束和导数
  2. 检查约束范数是否低于阈值（`MNEE_SOLVER_THRESHOLD = 0.001`）
  3. 求解线性方程组获得位移
  4. 在切平面上移动顶点并投影回原始表面
  5. 验证路径仍然是透射的
  6. 若投影失败则减小步长（自适应步长控制）

### mnee_sample_bsdf_dh()

- **签名**: `ccl_device_forceinline float2 mnee_sample_bsdf_dh(ClosureType type, const float alpha_x, const float alpha_y, const float sample_u, const float sample_v)`
- **功能**: 在半向量度量下采样微面元法线分布。支持 GGX 和 Beckmann 分布，以及各向异性粗糙度。返回法线在切平面上的投影分量。

### mnee_eval_bsdf_contribution()

- **签名**: `ccl_device_forceinline Spectrum mnee_eval_bsdf_contribution(KernelGlobals kg, ccl_private ShaderClosure *closure, const float3 wi, const float3 wo)`
- **功能**: 在解的界面处计算 BSDF 贡献。利用采样 PDF 与 BSDF 的关系进行化简：`contribution = (1 - F) * G * |h.wi / (n.wi * n.h^2)|`。

### mnee_compute_transfer_matrix()

- **签名**: `ccl_device_forceinline bool mnee_compute_transfer_matrix(const ccl_private ShaderData *sd, const ccl_private LightSample *ls, const bool light_fixed_direction, const int vertex_count, ccl_private ManifoldVertex *vertices, ccl_private float *dx1_dxlight, ccl_private float *dh_dx)`
- **功能**: 计算传递矩阵行列式 `|T1| = |dx1/dxn|` 和 `|dh/dx|`，用于广义几何项的计算。使用简化的块三对角 LU 分解。

### mnee_path_contribution()

- **签名**: `ccl_device_forceinline bool mnee_path_contribution(KernelGlobals kg, IntegratorState state, ccl_private ShaderData *sd, ccl_private ShaderData *sd_mnee, ccl_private LightSample *ls, const bool light_fixed_direction, const int vertex_count, ccl_private ManifoldVertex *vertices, ccl_private BsdfEval *throughput)`
- **功能**: 计算完整 MNEE 路径的贡献。包括：接收体 BSDF 评估、光源着色器评估、传递矩阵广义几何项、每个折射界面的镜面反射率评估，以及沿路径的可见性测试。

### kernel_path_mnee_sample()

- **签名**: `ccl_device_forceinline int kernel_path_mnee_sample(KernelGlobals kg, IntegratorState state, ccl_private ShaderData *sd, ccl_private ShaderData *sd_mnee, const ccl_private RNGState *rng_state, ccl_private LightSample *ls, ccl_private BsdfEval *throughput)`
- **功能**: MNEE 路径采样的顶层入口。三个阶段：
  1. 发送种子射线从着色点到光源，收集焦散投射体上的流形顶点
  2. 在镜面流形上执行牛顿迭代找到满足斯涅尔定律的解
  3. 若解存在，计算对应路径的贡献
- **返回值**: 焦散投射体的数量（0 表示失败）。

## 依赖关系

- **内部头文件**:
  - `kernel/geom/motion_triangle.h` - 运动三角形顶点和法线
  - `kernel/geom/shader_data.h` - 着色数据设置
  - `kernel/geom/triangle.h` - 三角形几何操作
  - `kernel/light/sample.h` - 光源采样和评估
- **被引用**:
  - `kernel/integrator/shade_surface.h` - 表面着色中调用 MNEE 采样

## 实现细节 / 关键算法

1. **ManifoldVertex 结构**: 包含位置(p)、切线(dp_du/dp_dv)、法线(n/ng)及其偏导数(dn_du/dn_dv)、UV 坐标、闭包信息(eta/bsdf)、约束向量和导数矩阵(a/b/c)。
2. **2x2 矩阵运算**: 通过 `float4` 编码行优先 2x2 矩阵，提供乘法、行列式和求逆操作。
3. **自适应步长**: 牛顿求解器从 `beta = 0.1` 开始，逐步增大到 1.0。若投影失败或路径不再透射，减半步长。步长低于 `MNEE_MINIMUM_STEP_SIZE = 0.0001` 时终止。
4. **最大限制常量**:
   - `MNEE_MAX_ITERATIONS = 64` - 最大牛顿迭代次数
   - `MNEE_MAX_CAUSTIC_CASTERS = 6` - 最大焦散投射体数量
   - `MNEE_MAX_INTERSECTION_COUNT = 10` - 种子射线最大交点数
5. **深度限制检查**: 在启动牛顿求解前，检查透射弹射深度、漫反射弹射深度和总弹射深度是否在限制内。
6. **平滑法线要求**: MNEE 要求折射表面具有平滑法线（`SHADER_SMOOTH_NORMAL`），因为流形探索依赖于表面上任意点可创建的局部微分几何。

## 关联文件

- `kernel/integrator/shade_surface.h` - 在表面着色的直接光照阶段调用 `kernel_path_mnee_sample`
- `kernel/integrator/intersect_closest.h` - MNEE 防火花路径剔除逻辑
- `kernel/integrator/shade_background.h` - MNEE 防火花背景光评估抑制
