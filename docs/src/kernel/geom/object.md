# object.h - 对象图元变换与属性查询

## 概述
本文件实现了 Cycles 渲染器中对象(Object)图元的核心功能，包括对象空间与世界空间之间的变换操作、运动模糊变换插值、对象属性查询(颜色、透明度、Pass ID、随机数等)以及层次包围体(BVH)实例变换。所有网格和曲线图元都属于某个对象，同一网格/曲线可以通过不同对象多次实例化。

## 核心函数

### 变换相关

#### object_fetch_transform()
- **签名**: `ccl_device_inline Transform object_fetch_transform(KernelGlobals kg, const int object, enum ObjectTransform type)`
- **功能**: 获取对象的正向或逆向变换矩阵(对象空间到世界空间或反向)。

#### object_fetch_transform_motion()
- **签名**: `ccl_device_inline Transform object_fetch_transform_motion(KernelGlobals kg, const int object, const float time)`
- **功能**: 获取运动模糊对象在给定时间的变换矩阵。通过 `transform_motion_array_interpolate()` 在分解变换(DecomposedTransform)数组中插值。受 `__OBJECT_MOTION__` 保护。

#### object_fetch_transform_motion_test()
- **签名**: `ccl_device_inline Transform object_fetch_transform_motion_test(KernelGlobals kg, const int object, const float time, Transform *itfm)`
- **功能**: 获取变换矩阵，自动检测对象是否有运动模糊标志。有运动模糊时插值计算，无运动模糊时直接返回静态变换。同时可输出逆变换。

#### object_get_transform() / object_get_inverse_transform()
- **签名**: `ccl_device_inline Transform object_get_transform(KernelGlobals kg, const ShaderData *sd)`
- **功能**: 从 ShaderData 获取当前对象的正向/逆向变换。运动模糊时使用 ShaderData 中缓存的 `ob_tfm_motion`，否则直接获取静态变换。

### 空间变换

#### object_position_transform()
- **签名**: `template<class T> ccl_device_inline void object_position_transform(KernelGlobals kg, const ShaderData *sd, T *P)`
- **功能**: 将位置从对象空间变换到世界空间。

#### object_inverse_position_transform()
- **签名**: `ccl_device_inline void object_inverse_position_transform(KernelGlobals kg, const ShaderData *sd, float3 *P)`
- **功能**: 将位置从世界空间变换到对象空间。

#### object_normal_transform() / object_inverse_normal_transform()
- **签名**: `ccl_device_inline void object_normal_transform(KernelGlobals kg, const ShaderData *sd, float3 *N)`
- **功能**: 法线的正向/逆向变换。法线变换使用逆转置矩阵以保持正确方向。

#### object_dir_transform() / object_inverse_dir_transform()
- **签名**: `ccl_device_inline void object_dir_transform(KernelGlobals kg, const ShaderData *sd, float3 *D)`
- **功能**: 方向向量的正向/逆向变换。

### 对象属性查询

#### object_location()
- **签名**: `ccl_device_inline float3 object_location(KernelGlobals kg, const ShaderData *sd)`
- **功能**: 获取对象中心位置(从变换矩阵的平移分量提取)。

#### object_color() / object_alpha() / object_pass_id()
- **功能**: 分别获取对象的颜色、透明度和 Pass ID。

#### object_random_number()
- **功能**: 获取对象随机数，用于着色器变化。

#### object_volume_density()
- **功能**: 获取体积密度值。

#### object_cryptomatte_id() / object_cryptomatte_asset_id()
- **功能**: 获取 Cryptomatte ID，用于合成中的对象/素材遮罩。

### 粒子属性

#### particle_index() / particle_age() / particle_lifetime() / particle_size() / particle_rotation() / particle_location() / particle_velocity() / particle_angular_velocity()
- **功能**: 查询生成当前对象的粒子系统相关数据。

### BVH 实例变换

#### bvh_instance_push()
- **签名**: `ccl_device_inline void bvh_instance_push(KernelGlobals kg, const int object, const Ray *ray, float3 *P, float3 *dir, float3 *idir)`
- **功能**: 将光线从世界空间变换到对象空间以进入 BVH 中的静态对象实例。同时计算方向的倒数(idir)用于 BVH 遍历优化。

#### bvh_instance_motion_push()
- **签名**: `ccl_device_inline void bvh_instance_motion_push(KernelGlobals kg, const int object, const Ray *ray, float3 *P, float3 *dir, float3 *idir)`
- **功能**: 运动模糊版本的 BVH 实例入栈。使用时间插值的变换矩阵。

#### bvh_instance_pop()
- **签名**: `ccl_device_inline void bvh_instance_pop(const Ray *ray, float3 *P, float3 *dir, float3 *idir)`
- **功能**: 退出 BVH 对象实例，恢复世界空间光线参数。

## 依赖关系
- **内部头文件**: `kernel/globals.h`, `kernel/types.h`
- **被引用**: 广泛被 geom 子模块(curve_intersect.h, triangle.h, triangle_intersect.h, volume.h, point.h, point_intersect.h, primitive.h, shader_data.h, motion_triangle_intersect.h)及 BVH、SVM、OSL、light、integrator 等模块引用(共 29 个文件)

## 实现细节 / 关键算法

### 运动模糊变换
运动模糊对象的变换通过分解变换(DecomposedTransform)数组进行插值(平移、旋转、缩放分别插值)。ShaderData 中缓存了 `ob_tfm_motion` 和 `ob_itfm_motion` 以避免重复计算。逆变换通过 `transform_inverse()` 实时计算。

### 法线变换
法线必须使用**逆转置矩阵**变换：对象到世界使用逆变换矩阵的转置(`transform_direction_transposed` 作用于逆矩阵)，世界到对象使用正向矩阵的转置。

### BVH 方向夹紧
`bvh_clamp_direction()` 将接近零的方向分量夹紧到极小值 `8.271806E-25f`(保持符号)，避免 BVH 遍历中的除零问题。

### 宏别名
`object_position_transform_auto`、`object_dir_transform_auto`、`object_normal_transform_auto` 目前直接别名为不带 `_auto` 的版本，预留给未来可能需要显式地址空间限定符的设备。

## 关联文件
- `kernel/geom/shader_data.h` - 着色数据初始化中广泛调用对象变换
- `kernel/bvh/bvh.h` - BVH 遍历中使用实例入栈/出栈
- `kernel/geom/triangle.h` - 三角形法线变换
- `kernel/geom/curve_intersect.h` - 曲线着色设置中的变换
