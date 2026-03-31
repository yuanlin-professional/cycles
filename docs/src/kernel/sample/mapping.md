# mapping.h - 采样空间映射与分布函数

## 概述
本文件提供将均匀分布的随机样本映射到各种几何域（单位圆盘、半球、球面、锥体、正多边形）上的采样函数，以及几何分布、指数分布等概率分布的采样方法。这些函数是路径追踪渲染器的基础采样工具，被着色器（BSDF 采样）、相机（景深）、光源采样等模块广泛使用。代码源自 Open Shading Language (OSL)，采用同心映射等技术保持分层性。

## 类与结构体
无。

## 枚举与常量
无独立定义的枚举或常量。函数内部使用 `M_PI_F`、`M_PI_2_F`、`M_PI_4_F`、`M_1_PI_F`、`M_1_2PI_F`、`M_2PI_F` 等数学常量。

## 核心函数

### sample_uniform_disk()
- **签名**: `ccl_device float2 sample_uniform_disk(const float2 rand)`
- **功能**: 将 `[0,1]^2` 上的二维均匀随机样本映射到单位圆盘 `[-1,1]^2` 上。使用 Shirley-Chiu 同心映射（concentric mapping），相比极坐标映射能更好地保持输入样本的分层性。该函数是多个半球/锥体采样函数的基础构件。

### make_orthonormals_tangent()
- **签名**: `ccl_device void make_orthonormals_tangent(const float3 N, const float3 T, float3 *a, float3 *b)`
- **功能**: 根据法线 `N` 和切线 `T` 构造正交切线/副切线坐标系。用于各向异性 BSDF 需要对齐切线方向的场景。

### make_orthonormals_safe_tangent()
- **签名**: `ccl_device void make_orthonormals_safe_tangent(const float3 N, const float3 T, float3 *a, float3 *b)`
- **功能**: `make_orthonormals_tangent` 的安全版本。当 `N` 与 `T` 近似平行导致叉积退化时，自动回退到基于 `N` 的通用正交基构造。

### sample_cos_hemisphere()
- **签名**: `ccl_device_inline void sample_cos_hemisphere(const float3 N, const float2 rand_in, float3 *wo, float *pdf)`
- **功能**: 在法线 `N` 定义的半球上进行余弦加权采样。先在单位圆盘上采样，然后投影到半球面。输出采样方向 `wo` 和对应的概率密度 `pdf = cos(theta) / pi`。这是漫反射 BSDF 最常用的采样策略。

### pdf_cos_hemisphere()
- **签名**: `ccl_device_inline float pdf_cos_hemisphere(const float3 N, const float3 D)`
- **功能**: 计算余弦加权半球采样在方向 `D` 处的 PDF 值。返回 `max(0, dot(N,D)) / pi`。

### sample_uniform_hemisphere()
- **签名**: `ccl_device_inline void sample_uniform_hemisphere(const float3 N, const float2 rand, float3 *wo, float *pdf)`
- **功能**: 在法线 `N` 定义的半球上进行均匀采样。PDF 恒为 `1/(2*pi)`。

### pdf_uniform_cone()
- **签名**: `ccl_device_inline float pdf_uniform_cone(const float3 N, const float3 D, const float angle)`
- **功能**: 计算以 `N` 为轴、张角为 `angle` 的锥体内均匀采样的 PDF。若方向 `D` 在锥体外则返回 0。

### sample_uniform_cone()
- **签名**: `ccl_device_inline float3 sample_uniform_cone(const float3 N, const float one_minus_cos_angle, const float2 rand, float *cos_theta, float *pdf)`
- **功能**: 在以 `N` 为轴的锥体内均匀采样方向。使用同心圆盘映射投影到锥面，保证关于立体角的均匀分布。参数传入 `1 - cos(angle)` 而非角度本身，以在小角度时减轻浮点精度问题。

### sample_uniform_sphere()
- **签名**: `ccl_device float3 sample_uniform_sphere(const float2 rand)`
- **功能**: 在单位球面上均匀采样方向。

### regular_polygon_sample()
- **签名**: `ccl_device float2 regular_polygon_sample(const float corners, float rotation, const float2 rand)`
- **功能**: 在具有指定角数和旋转角的正多边形内均匀采样点。用于相机景深模拟多边形光圈形状（如五边形、六边形光圈）。

### sample_geometric_distribution()
- **签名**: `ccl_device_inline int sample_geometric_distribution(const float rand, const float r, float &pmf, const int cut_off = INT_MAX)`
- **功能**: 从几何分布 `p(x) = r * (1-r)^x` 中采样整数随机变量，同时计算概率质量函数值。支持截断上限 `cut_off`。用于随机游走等需要离散步数采样的场景。

### sample_exponential_distribution() [标量版]
- **签名**: `ccl_device_inline float sample_exponential_distribution(const float rand, const float lambda)`
- **功能**: 从速率参数为 `lambda` 的指数分布中采样。使用逆变换采样法：`-log(1-rand) / lambda`。

### sample_exponential_distribution() [有界版]
- **签名**: `ccl_device_inline float sample_exponential_distribution(const float rand, const float lambda, const Interval<float> t)`
- **功能**: 从有界指数分布中采样，结果限制在区间 `(t.min, t.max)` 内。用于体积渲染中在有限距离区间内采样散射事件。

### pdf_exponential_distribution()
- **签名**: `ccl_device_inline Spectrum pdf_exponential_distribution(const float x, const Spectrum lambda, const Interval<float> t)`
- **功能**: 计算有界指数分布在位置 `x` 处的 PDF。支持光谱类型 `Spectrum` 的速率参数（多通道体积消光系数）。

## 依赖关系
- **内部头文件**: `util/math.h`、`util/projection.h`
- **被引用**:
  - `src/kernel/svm/ao.h` - 环境光遮蔽采样
  - `src/kernel/light/common.h` - 光源采样通用函数
  - `src/kernel/camera/camera.h` - 相机景深/运动模糊
  - `src/kernel/closure/bsdf_ashikhmin_velvet.h` - Ashikhmin 天鹅绒 BSDF
  - `src/kernel/closure/bsdf_burley.h` - Burley 漫反射 BSDF
  - `src/kernel/closure/bsdf_diffuse_ramp.h` - 渐变漫反射 BSDF
  - `src/kernel/closure/bsdf_diffuse.h` - 标准漫反射 BSDF
  - `src/kernel/closure/bsdf_microfacet.h` - 微表面 BSDF（GGX 等）
  - `src/kernel/closure/bsdf_oren_nayar.h` - Oren-Nayar BSDF
  - `src/kernel/closure/bsdf_sheen.h` - 光泽布料 BSDF
  - `src/kernel/closure/bsdf_toon.h` - 卡通 BSDF

## 实现细节 / 关键算法

### Shirley-Chiu 同心圆盘映射
`sample_uniform_disk` 实现了经典的 Shirley 同心映射：将正方形 `[-1,1]^2` 分为四个扇区，每个扇区映射到圆盘的四分之一。相比朴素的极坐标映射 `(r, theta) = (sqrt(u), 2*pi*v)`，同心映射避免了极点附近的样本聚集，能更好地保持低差异序列的分层特性。

### 半球采样的圆盘投影
`sample_cos_hemisphere` 和 `sample_uniform_hemisphere` 都先在圆盘上采样，再计算对应的 z 坐标投影到半球。这种方法天然继承了同心映射的分层优势：
- 余弦加权：`z = sqrt(1 - r^2)`，圆盘上均匀分布直接对应半球上的余弦加权分布
- 均匀半球：需要额外的半径重映射 `xy *= sqrt(z+1)` 来抵消投影带来的密度变化

### 锥体采样的半径重映射
`sample_uniform_cone` 的核心创新在于半径重映射公式的推导：从余弦加权半球采样的圆盘半径 `r_disk` 反推到均匀锥体采样的半径 `r_cone`。具体推导过程为：先求 `r_disk` 到均匀随机数的逆映射 `rand = r_disk^2`，再代入锥体的 CDF 逆函数，得到最终映射 `cos_theta = 1 - r^2 * (1-cos_angle)`。

### 正多边形采样
`regular_polygon_sample` 将多边形分解为等腰三角形，通过均匀三角形采样（`u = sqrt(u), v = v*u`）加旋转组合实现。该方法用于模拟相机的多边形光圈散景效果。

## 关联文件
- `src/kernel/sample/pattern.h` - 采样模式调度，提供随机数输入
- `src/kernel/closure/bsdf_*.h` - 各类 BSDF 闭包，调用半球/圆盘采样
- `src/kernel/camera/camera.h` - 相机模块，调用正多边形采样生成景深
- `src/kernel/light/common.h` - 光源采样通用函数
- `src/util/projection.h` - 提供 `polar_to_cartesian` 等投影工具
