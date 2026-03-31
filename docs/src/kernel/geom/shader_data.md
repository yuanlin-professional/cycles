# shader_data.h - 着色数据(ShaderData)初始化

## 概述
本文件实现了 Cycles 渲染器中着色数据(ShaderData)结构的初始化功能。ShaderData 是着色管线的核心数据结构，包含交点位置、法线、UV、微分等所有着色所需信息。提供了从入射光线交点、网格采样位置、位移(displacement)、曲线和背景/体积等多种来源初始化 ShaderData 的函数。

## 核心函数

### shader_setup_object_transforms()
- **签名**: `ccl_device void shader_setup_object_transforms(KernelGlobals kg, ShaderData *sd, const float time)`
- **功能**: 为运动模糊对象设置变换矩阵。将时间插值的正向和逆向变换缓存到 `sd->ob_tfm_motion` 和 `sd->ob_itfm_motion`。受 `__OBJECT_MOTION__` 保护。

### shader_setup_from_ray()
- **签名**: `ccl_device_inline void shader_setup_from_ray(KernelGlobals kg, ShaderData *sd, const Ray *ray, const Intersection *isect)`
- **功能**: 从光线交点初始化 ShaderData(最常用的入口)。处理流程：
  1. 将 Intersection 数据(u, v, t, type, object, prim)写入 ShaderData
  2. 设置时间和运动变换
  3. 根据图元类型分发着色设置:
     - `PRIMITIVE_CURVE`: 调用 `curve_shader_setup()`
     - `PRIMITIVE_POINT`: 调用 `point_shader_setup()`
     - `PRIMITIVE_TRIANGLE`: 调用 `triangle_shader_setup()`
     - `PRIMITIVE_MOTION_TRIANGLE`: 调用 `motion_triangle_shader_setup()`
  4. 非曲线/点的图元执行实例变换(法线、dPdu/dPdv)
  5. 设置着色器标志
  6. 背面检测并翻转法线
  7. 计算光线微分(dP, dI, du, dv)

### shader_setup_from_sample()
- **签名**: `ccl_device_inline void shader_setup_from_sample(KernelGlobals kg, ShaderData *sd, const float3 P, const float3 Ng, const float3 I, const int shader, const int object, const int prim, const float u, const float v, const float t, const float time, const bool object_space, const bool is_lamp)`
- **功能**: 从网格采样位置初始化 ShaderData。用于光源采样等场景。处理对象空间到世界空间的变换、平滑法线计算和 dPdu/dPdv。不设置光线微分(全部为零)。

### shader_setup_from_displace()
- **签名**: `ccl_device void shader_setup_from_displace(KernelGlobals kg, ShaderData *sd, const int object, const int prim, const float u, const float v)`
- **功能**: 为位移贴图初始化 ShaderData。从三角形获取位置和法线，强制启用平滑法线(`SHADER_SMOOTH_NORMAL`)，将入射方向设为法线方向以避免除零。

### shader_setup_from_curve()
- **签名**: `ccl_device void shader_setup_from_curve(KernelGlobals kg, ShaderData *sd, const int object, const int prim, const int segment, const float u)`
- **功能**: 为曲线上的点初始化 ShaderData。设置曲线类型为粗曲线(`PRIMITIVE_CURVE_THICK`)，通过 Catmull-Rom 或线性基函数评估位置和切线。选择任意视线方向(避免 NaN)。受 `__HAIR__` 保护。

### shader_setup_from_background()
- **签名**: `ccl_device_inline void shader_setup_from_background(KernelGlobals kg, ShaderData *sd, const float3 ray_P, const float3 ray_D, const float ray_time)`
- **功能**: 为背景着色初始化 ShaderData。设置 P 为光线方向(非位置)，N/Ng/wi 为光线方向的反向，ray_length 为 FLT_MAX，无图元/对象关联。

### shader_setup_from_volume()
- **签名**: `ccl_device_inline void shader_setup_from_volume(ShaderData *sd, const Ray *ray, const int object)`
- **功能**: 为体积内部的点初始化 ShaderData。位置设为 `ray->P + ray->D * ray->tmin`，类型为 `PRIMITIVE_VOLUME`。受 `__VOLUME__` 保护。

## 依赖关系
- **内部头文件**: `kernel/globals.h`, `kernel/geom/curve_intersect.h`, `kernel/geom/motion_triangle_shader.h`, `kernel/geom/object.h`, `kernel/geom/point_intersect.h`, `kernel/geom/triangle_intersect.h`, `kernel/util/differential.h`
- **被引用**: `integrator/volume_shader.h`, `integrator/shade_volume.h`, `integrator/shade_background.h`, `integrator/shade_shadow.h`, `integrator/shade_surface.h`, `integrator/intersect_volume_stack.h`, `integrator/mnee.h`, `light/sample.h`, `osl/services.cpp`, `bake/bake.h`

## 实现细节 / 关键算法

### 图元类型分发
`shader_setup_from_ray()` 是渲染管线中最关键的着色设置函数，通过图元类型标志进行多路分发：
- 曲线和点云有各自独立的着色设置(包含法线计算和变换)
- 三角形和运动三角形在着色设置后还需额外的实例变换步骤

### 背面检测
通过 `dot(Ng, wi) < 0` 检测背面。命中背面时翻转 Ng、N 和 dPdu/dPdv，并设置 `SD_BACKFACING` 标志。

### 光线微分
使用紧凑的微分传播：
- `dP`: 通过 `differential_transfer_compact()` 从光线微分传递到表面
- `dI`: 通过 `differential_incoming_compact()` 计算入射方向微分
- `du/dv`: 通过 `differential_dudv_compact()` 从 dP 和 dPdu/dPdv 计算参数微分

## 关联文件
- `kernel/geom/curve_intersect.h` - 曲线着色设置 `curve_shader_setup()`
- `kernel/geom/triangle_intersect.h` - 三角形着色设置 `triangle_shader_setup()`
- `kernel/geom/point_intersect.h` - 点云着色设置 `point_shader_setup()`
- `kernel/geom/motion_triangle_shader.h` - 运动三角形着色设置
- `kernel/geom/object.h` - 对象变换操作
- `kernel/util/differential.h` - 微分计算工具
