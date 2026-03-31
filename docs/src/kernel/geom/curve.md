# curve.h - 曲线图元着色数据与属性读取

## 概述
本文件实现了 Cycles 渲染器中曲线图元(用于渲染毛发和毛皮)的着色相关功能。提供了曲线属性读取与插值、曲线厚度计算、切线法线计算以及曲线包围盒(bounds)工具函数。曲线可以作为平面飘带(ribbon)或带实际厚度的曲线渲染，也可以作为线段渲染以提高性能。

## 核心函数

### curve_attribute\<T\>()
- **签名**: `template<typename T> ccl_device dual<T> curve_attribute(KernelGlobals kg, const ccl_private ShaderData *sd, const AttributeDescriptor desc, const bool dx = false, const bool dy = false)`
- **功能**: 读取曲线元素上的属性值并计算偏微分。对于 `ATTR_ELEMENT_CURVE_KEY` 类型，在两个控制点之间基于参数 u 进行线性插值；对于 `ATTR_ELEMENT_CURVE` 类型，直接读取每条曲线的属性值。

### curve_attribute_dfdx() / curve_attribute_dfdy()
- **签名**: `template<typename T> ccl_device_inline T curve_attribute_dfdx(const differential &du, const T &f0, const T &f1)`
- **功能**: 计算曲线属性关于 x/y 方向的偏微分。基于链式法则 df/dx = df/du * du/dx = (f1 - f0) * du.dx。

### curve_thickness()
- **签名**: `ccl_device float curve_thickness(KernelGlobals kg, const ccl_private ShaderData *sd)`
- **功能**: 计算曲线在当前着色点处的厚度(直径)。通过在两个控制点的半径之间线性插值得到当前半径 r，返回 2r。支持运动模糊(motion blur)场景下的关键帧插值。

### curve_random()
- **签名**: `ccl_device float curve_random(KernelGlobals kg, const ccl_private ShaderData *sd)`
- **功能**: 获取曲线的随机数属性(`ATTR_STD_CURVE_RANDOM`)，用于着色器变化。

### curve_motion_center_location()
- **签名**: `ccl_device float3 curve_motion_center_location(KernelGlobals kg, const ccl_private ShaderData *sd)`
- **功能**: 计算曲线运动模糊通道(motion pass)的中心位置，在两个控制点之间线性插值，忽略半径。

### curve_tangent_normal()
- **签名**: `ccl_device float3 curve_tangent_normal(const ccl_private ShaderData *sd)`
- **功能**: 计算曲线的切线法线方向，用于着色。通过从入射方向中减去切线分量并归一化得到。

### curvebounds()
- **签名**: `ccl_device_inline void curvebounds(float *lower, float *upper, float *extremta, float *extrema, float *extremtb, float *extremb, float p0, float p1, float p2, float p3)`
- **功能**: 计算三次曲线在 [0,1] 参数范围内的包围盒(上下界)和极值点。通过求解二次方程找到极值点位置。

## 依赖关系
- **内部头文件**: `kernel/globals.h`, `kernel/geom/attribute.h`, `kernel/geom/motion_curve.h`
- **被引用**: `geom/primitive.h`, `svm/geometry.h`, `svm/closure.h`, `osl/services_gpu.h`

## 实现细节 / 关键算法
- **曲线属性插值**: 曲线控制点属性通过线性插值获取，段号由 `PRIMITIVE_UNPACK_SEGMENT(sd->type)` 解包得到，用于确定当前段的两个控制点索引 k0 和 k1。
- **运动模糊支持**: 曲线厚度计算检测 `PRIMITIVE_MOTION` 标志，若为运动曲线则通过 `motion_curve_keys_linear()` 获取运动插值后的控制点。
- **包围盒计算**: `curvebounds()` 将三次多项式 p(t) = p3*t^3 + p2*t^2 + p1*t + p0 的极值点通过求解其导数的根(二次方程)来确定，从而计算紧凑的包围盒。
- 整个文件被 `__HAIR__` 预处理宏保护，仅在毛发功能启用时编译。

## 关联文件
- `kernel/geom/curve_intersect.h` - 曲线光线相交测试
- `kernel/geom/motion_curve.h` - 运动模糊曲线关键帧插值
- `kernel/geom/attribute.h` - 属性查找与读取基础设施
- `kernel/geom/primitive.h` - 统一图元属性接口
