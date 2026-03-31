# projection.h - 全景相机投影模型与方向映射函数库

## 概述
本文件实现了 Cycles 渲染器中全景相机所需的各种投影模型，提供了**二维纹理坐标 (u, v) 与三维笛卡尔方向向量之间的双向映射**。文件同时包含球面立体声 (Spherical Stereo) 变换，用于 VR 立体渲染。作为相机子系统的底层数学工具库，它被 `camera.h` 直接依赖，不涉及光线生成逻辑，只负责纯粹的几何投影变换。

代码部分改编自 Open Shading Language (OSL)，采用 BSD-3-Clause 许可证。

## 类与结构体
本文件无自定义结构体或类定义。所有函数操作的核心数据结构为 `KernelCamera`（定义于 `kernel/types.h`），通过 `ccl_constant` 指针以只读方式访问相机参数。

## 枚举与常量
本文件引用以下枚举（定义于 `kernel/types.h`）：

| 枚举值 | 说明 |
|---|---|
| `PANORAMA_EQUIRECTANGULAR` (0) | 等距矩形投影 |
| `PANORAMA_FISHEYE_EQUIDISTANT` (1) | 鱼眼等距投影 |
| `PANORAMA_FISHEYE_EQUISOLID` (2) | 鱼眼等立体角投影 |
| `PANORAMA_MIRRORBALL` (3) | 镜面球投影 |
| `PANORAMA_FISHEYE_LENS_POLYNOMIAL` (4) | 鱼眼多项式镜头模型 |
| `PANORAMA_EQUIANGULAR_CUBEMAP_FACE` (5) | 等角立方体贴图面投影 |
| `PANORAMA_CENTRAL_CYLINDRICAL` (6) | 中心柱面投影 |

## 核心函数

### 等距矩形投影 (Equirectangular)

#### direction_to_equirectangular_range()
- **签名**: `ccl_device float2 direction_to_equirectangular_range(const float3 dir, const float4 range)`
- **功能**: 将三维方向向量转换为等距矩形投影下的 (u, v) 坐标。`range` 参数指定经度/纬度的偏移与缩放范围。零向量输入返回 `(0, 0)`。

#### equirectangular_range_to_direction()
- **签名**: `ccl_device float3 equirectangular_range_to_direction(const float u, const float v, const float4 range)`
- **功能**: 等距矩形投影坐标的逆变换，将 (u, v) 转换回三维方向向量。内部调用 `spherical_to_direction()` 完成球坐标到笛卡尔坐标的转换。

#### direction_to_equirectangular()
- **签名**: `ccl_device float2 direction_to_equirectangular(const float3 dir)`
- **功能**: 使用默认完整范围 (经度 -2pi..0, 纬度 0..pi) 的等距矩形投影方向映射。

#### equirectangular_to_direction()
- **签名**: `ccl_device float3 equirectangular_to_direction(const float u, const float v)`
- **功能**: 默认完整范围下的等距矩形投影逆映射。

### 中心柱面投影 (Central Cylindrical)

#### direction_to_central_cylindrical()
- **签名**: `ccl_device float2 direction_to_central_cylindrical(const float3 dir, const float4 range)`
- **功能**: 将方向向量映射到中心柱面投影坐标。使用 `inverse_lerp` 将角度和 z 值归一化到 `range` 指定的范围内。

#### central_cylindrical_to_direction()
- **签名**: `ccl_device float3 central_cylindrical_to_direction(const float u, const float v, const float4 range)`
- **功能**: 中心柱面投影的逆映射。使用 `mix` 在范围内插值还原角度和 z 分量。

### 鱼眼投影 (Fisheye)

#### fisheye_to_direction()
- **签名**: `ccl_device_inline float3 fisheye_to_direction(const float theta, const float u, float v, const float r)`
- **功能**: 所有鱼眼投影模型的共享内部函数。根据入射角 `theta` 和图像平面上的归一化坐标 `(u, v)` 及其径向距离 `r`，计算对应的三维方向向量。

#### direction_to_fisheye_equidistant() / fisheye_equidistant_to_direction()
- **签名**: `float2 direction_to_fisheye_equidistant(const float3 dir, const float fov)` / `float3 fisheye_equidistant_to_direction(float u, float v, float fov)`
- **功能**: 等距鱼眼投影的双向映射。`fov` 为视场角。当径向距离 `r > 1.0` 时返回零向量（表示超出视野范围）。

#### direction_to_fisheye_equisolid() / fisheye_equisolid_to_direction()
- **签名**: `float2 direction_to_fisheye_equisolid(const float3 dir, const float lens, const float width, const float height)` / `float3 fisheye_equisolid_to_direction(float u, float v, float lens, const float fov, const float width, const float height)`
- **功能**: 等立体角鱼眼投影的双向映射。映射公式为 `r = 2f * sin(theta/2)`，其中 `f` 为焦距 (`lens`)，`width`/`height` 为传感器尺寸。超过最大径向距离 `rmax` 时返回零向量。

#### fisheye_lens_polynomial_to_direction()
- **签名**: `ccl_device_inline float3 fisheye_lens_polynomial_to_direction(float u, float v, float coeff0, const float4 coeffs, const float fov, const float width, const float height)`
- **功能**: 多项式镜头失真模型的正向映射。使用四阶多项式 `theta = -(coeff0 + c1*r + c2*r^2 + c3*r^3 + c4*r^4)` 计算入射角。超出 `fov/2` 时返回零向量。

#### direction_to_fisheye_lens_polynomial()
- **签名**: `ccl_device float2 direction_to_fisheye_lens_polynomial(float3 dir, const float coeff0, const float4 coeffs, const float width, const float height)`
- **功能**: 多项式镜头失真模型的逆映射。由于多项式无解析逆，采用**牛顿迭代法**求解：以线性近似作为初始值，最多迭代 20 次，收敛阈值为 `1e-6`。

### 镜面球投影 (Mirror Ball)

#### mirrorball_to_direction()
- **签名**: `ccl_device float3 mirrorball_to_direction(const float u, const float v)`
- **功能**: 将镜面球 (Chrome Ball) 纹理坐标映射为方向向量。原理为：先将 (u,v) 映射到单位球表面点，再计算该点对 Y 轴负方向入射光的反射方向。超出球面范围时返回零向量。

#### direction_to_mirrorball()
- **签名**: `ccl_device float2 direction_to_mirrorball(float3 dir)`
- **功能**: `mirrorball_to_direction()` 的逆函数，将方向向量转换回镜面球纹理坐标。

### 等角立方体贴图面投影 (Equiangular Cubemap Face)

#### equiangular_cubemap_face_to_direction()
- **签名**: `ccl_device float3 equiangular_cubemap_face_to_direction(float u, float v)`
- **功能**: 将单个等角立方体贴图面的 (u,v) 坐标转换为方向。使用 `tan()` 函数实现等角采样分布。参考 Google VR 视频投影技术。

#### direction_to_equiangular_cubemap_face()
- **签名**: `ccl_device float2 direction_to_equiangular_cubemap_face(const float3 dir)`
- **功能**: 等角立方体贴图面的逆映射，使用 `atan2()` 将方向向量映射回贴图坐标。

### 全景投影调度器

#### panorama_to_direction()
- **签名**: `ccl_device_inline float3 panorama_to_direction(ccl_constant KernelCamera *cam, const float u, float v)`
- **功能**: **全景投影的统一入口函数**。根据 `cam->panorama_type` 枚举值分发到具体的投影模型。被 `camera.h` 中的 `camera_panorama_direction()` 调用。

#### direction_to_panorama()
- **签名**: `ccl_device_inline float2 direction_to_panorama(ccl_constant KernelCamera *cam, const float3 dir)`
- **功能**: **全景逆投影的统一入口函数**。根据相机全景类型将方向向量映射回纹理坐标。被 `camera.h` 中的 `camera_world_to_ndc()` 调用。

### 球面立体声变换

#### spherical_stereo_transform()
- **签名**: `ccl_device_inline void spherical_stereo_transform(ccl_constant KernelCamera *cam, ccl_private float3 *P, ccl_private float3 *D)`
- **功能**: 对光线原点和方向施加球面立体偏移，用于 VR 立体渲染。根据 `interocular_offset`（双眼间距）横向偏移光线原点，并根据 `convergence_distance`（会聚距离）调整方向使双眼光线在指定距离处会聚。支持**极点融合** (pole merge)：当方向向量接近天顶/天底时，通过余弦淡出逐步减小双眼偏移，避免极点区域出现立体伪影。会聚距离为 `FLT_MAX` 时为平行会聚模式（不修改方向）。

## 依赖关系
- **内部头文件**:
  - `kernel/types.h` — 提供 `KernelCamera` 结构体和 `PanoramaType` 枚举定义
  - `util/math.h` — 数学工具函数（`atan2f`, `acosf`, `sinf`, `tanf`, `normalize`, `dot`, `cross` 等）
  - `util/types.h` — 基础类型定义（`float2`, `float3`, `float4`）
- **被引用**:
  - `src/kernel/camera/camera.h` — 全景相机光线生成与 NDC 坐标转换
  - `src/kernel/svm/image.h` — 环境贴图纹理采样
  - `src/kernel/light/background.h` — 环境光采样
  - `src/kernel/geom/primitive.h` — 几何图元相关
  - `src/kernel/bake/bake.h` — 烘焙功能
  - `src/test/kernel_camera_projection_test.cpp` — 单元测试

## 实现细节 / 关键算法

### 多项式镜头逆映射的牛顿迭代
`direction_to_fisheye_lens_polynomial()` 需要求解多项式方程 `theta = coeff0 + c1*r + c2*r^2 + c3*r^3 + c4*r^4` 中的 `r`。由于四阶多项式无通用解析解，使用牛顿法迭代：
1. **初始值**: 假设高阶系数为零，取线性近似解 `r = (theta - coeff0) / c1`
2. **迭代**: 计算 `F(r) = theta - model(r)` 及其导数 `F'(r)`，执行牛顿步 `r += F(r)/F'(r)`
3. **终止条件**: 变化量 `|r_new - r_old| < 1e-6` 或达到 20 次迭代

### 球面立体声的极点融合
为避免 VR 渲染在天顶/天底区域出现立体伪影，当视线方向的仰角在 `pole_merge_angle_from` 与 `pole_merge_angle_to` 之间时，对双眼偏移量施加余弦衰减 `fade = cos(fac * pi/2)`，使偏移平滑过渡到零。

### 零向量作为采样失败标记
多个投影函数在输入超出有效范围时返回零向量 `(0,0,0)`。调用方（如 `camera_sample_panorama()`）通过 `is_zero()` 检测此情况并返回零光谱，跳过该光线样本。

## 关联文件
- `src/kernel/camera/camera.h` — 使用本文件提供的投影函数生成全景相机光线
- `src/kernel/types.h` — `KernelCamera` 结构体与 `PanoramaType` 枚举定义
- `src/kernel/light/background.h` — 环境光方向映射使用相同投影函数
- `src/kernel/svm/image.h` — 环境贴图采样使用投影坐标转换
- `src/test/kernel_camera_projection_test.cpp` — 投影函数的单元测试
