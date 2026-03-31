# point.h - 点光源采样与求交

## 概述
本文件实现了 Cycles 渲染器中点光源的采样、求交和 PDF 计算。支持两种几何模式：球体光源（物理精确的球面采样）和传统点光源（基于朝向圆盘的近似）。球体模式下区分着色点在球体外部和内部两种情况，内部时支持基于透射的全球面采样。

## 核心函数

### point_light_sample()
- **签名**: `ccl_device_inline bool point_light_sample(KernelGlobals kg, const ccl_global KernelLight *klight, const float2 rand, const float3 P, const float3 N, const int shader_flags, ccl_private LightSample *ls)`
- **功能**: 在点光源上采样一个点。球体模式下：外部使用均匀锥体采样（锥角由球体张角决定），内部使用余弦半球采样或均匀球面采样（取决于是否有透射 BSDF）。通过余弦定理计算交点距离，并将采样点重映射到球面以避免小半径时的精度问题。传统模式下：在朝向着色点的圆盘上采样。

### sphere_light_pdf()
- **签名**: `ccl_device_forceinline float sphere_light_pdf(const float d_sq, const float r_sq, const float3 N, const float3 D, const uint32_t path_flag)`
- **功能**: 计算球体光源的采样 PDF。外部时为锥体均匀采样的 PDF，内部时根据是否有透射返回均匀球面或余弦半球的 PDF。

### point_light_mnee_sample_update()
- **签名**: `ccl_device_forceinline void point_light_mnee_sample_update(KernelGlobals kg, const ccl_global KernelLight *klight, ccl_private LightSample *ls, const float3 P, const float3 N, const uint32_t path_flag)`
- **功能**: 为 MNEE（多项式牛顿显式估计）更新点光源采样数据。球体模式下计算立体角到面积的雅可比行列式以保持面积度量的 PDF，传统模式下使用 `eval_fac * 4*pi` 作为面积 PDF。

### point_light_intersect()
- **签名**: `ccl_device_inline bool point_light_intersect(const ccl_global KernelLight *klight, const ccl_private Ray *ccl_restrict ray, ccl_private float *t)`
- **功能**: 光线与点光源的求交测试。零半径光源不可相交。球体模式使用 `ray_sphere_intersect`，传统模式使用 `ray_disk_intersect`（圆盘朝向射线起点）。

### point_light_sample_from_intersection()
- **签名**: `ccl_device_inline bool point_light_sample_from_intersection(KernelGlobals kg, const ccl_global KernelLight *klight, const float3 ray_P, const float3 ray_D, const float3 N, const uint32_t path_flag, ccl_private LightSample *ccl_restrict ls)`
- **功能**: 从光线求交结果构建点光源采样数据。球体模式使用 `sphere_light_pdf`，传统模式使用面积到立体角转换。

### point_light_tree_parameters()
- **签名**: `template<bool in_volume_segment> ccl_device_forceinline bool point_light_tree_parameters(const ccl_global KernelLight *klight, const float3 centroid, const float3 P, ccl_private float &cos_theta_u, ccl_private float2 &distance, ccl_private float3 &point_to_centroid)`
- **功能**: 为光源树的重要性计算提供点光源参数。球体模式下：外部计算等效圆盘的张角，内部设为全球面（cos_theta_u = -1）并使用半径/sqrt(2)作为距离。传统模式下计算斜边距离和张角。

## 依赖关系
- **内部头文件**: `kernel/globals.h`, `kernel/geom/object.h`, `kernel/light/common.h`, `util/math_intersect.h`
- **被引用**: `kernel/light/light.h`, `kernel/light/spot.h`, `kernel/light/tree.h`

## 实现细节 / 关键算法
1. **球体光源采样**: 当着色点在球体外部时，使用均匀锥体采样。锥角由 `sin^2(theta) = r^2/d^2` 确定。采样后通过余弦定理 `t = d*cos(theta) - sqrt(r^2 - d^2 + d^2*cos^2(theta))` 计算交点距离，`copysignf` 处理内外两种情况。最后将交点重映射到精确球面上防止小半径时的浮点精度问题。
2. **内部球体采样**: 着色点在球体内部时，根据表面是否有透射分量选择采样策略——有透射则均匀球面采样（覆盖所有方向），否则余弦半球采样（仅表面法线方向）。
3. **传统点光源**: 使用面向着色点的圆盘近似，半径由用户设置。PDF 为面积均匀分布经面积到立体角转换。这种模式不如球体模式物理精确，但兼容旧版行为。
4. **纹理坐标**: 使用逆变换矩阵将光源法线转到局部空间，再通过 `map_to_sphere` 映射到球面坐标作为 UV。

## 关联文件
- `kernel/light/common.h` - 提供 `LightSample`、`disk_light_sample`、`light_pdf_area_to_solid_angle`
- `kernel/light/light.h` - 调用 `point_light_sample`、`point_light_intersect`、`point_light_sample_from_intersection`
- `kernel/light/spot.h` - 聚光灯的求交复用 `point_light_intersect`
- `kernel/light/tree.h` - 调用 `point_light_tree_parameters`
- `util/math_intersect.h` - 提供 `ray_sphere_intersect` 和 `ray_disk_intersect`
