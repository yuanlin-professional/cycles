# triangle.h - 三角形图元着色数据与属性读取

## 概述
本文件实现了 Cycles 渲染器中三角形图元(Triangle Primitive)的核心着色功能。三角形是表示网格表面的基本图元，由三个顶点定义。提供了法线计算、平滑法线插值、顶点位置获取、偏微分(dPdu/dPdv)计算以及三角形属性读取与插值等功能。为了加速层次包围体(BVH)光线相交，使用了预计算的三角形存储结构。

## 核心函数

### triangle_interpolate\<T\>()
- **签名**: `template<typename T> ccl_device_inline T triangle_interpolate(const float u, const float v, const T f0, const T f1, const T f2)`
- **功能**: 通用的重心坐标插值。公式: `(1 - u - v) * f0 + u * f1 + v * f2`。用于在三角形顶点间插值任意量。

### triangle_normal()
- **签名**: `ccl_device_inline float3 triangle_normal(KernelGlobals kg, ShaderData *sd)`
- **功能**: 计算三角形的几何法线(面法线)。通过两条边的叉积得到，考虑负缩放时反转叉积顺序以保持正确朝向。

### triangle_point_normal()
- **签名**: `ccl_device_inline void triangle_point_normal(KernelGlobals kg, const int object, const int prim, const float u, const float v, float3 *P, float3 *Ng, int *shader)`
- **功能**: 同时计算三角形在指定重心坐标处的位置、几何法线和着色器 ID。用于光源采样和位移设置。

### triangle_vertices() / triangle_vertices_and_normals()
- **签名**: `ccl_device_inline void triangle_vertices(KernelGlobals kg, const int prim, float3 P[3])`
- **功能**: 获取三角形的三个顶点位置。`triangle_vertices_and_normals()` 版本同时获取顶点法线。

### triangle_smooth_normal()
- **签名**: `ccl_device_inline float3 triangle_smooth_normal(KernelGlobals kg, const float3 Ng, const int prim, const float u, float v)`
- **功能**: 通过重心坐标在三个顶点法线间插值计算平滑法线。若结果为零向量则回退到几何法线 Ng。提供三个重载版本：
  1. 基础版: 仅返回中心法线
  2. 带微分版: 额外输出 x/y 方向偏移处的法线(用于凹凸贴图)
  3. 非归一化版: 返回未归一化的插值法线(用于特殊计算)

### triangle_smooth_normal_unnormalized()
- **签名**: `ccl_device_inline float3 triangle_smooth_normal_unnormalized(KernelGlobals kg, const ShaderData *sd, const float3 Ng, const int prim, const float u, float v)`
- **功能**: 返回未归一化的平滑法线，确保法线在对象空间中。若变换已应用则先逆变换回对象空间。

### triangle_dPdudv()
- **签名**: `ccl_device_inline void triangle_dPdudv(KernelGlobals kg, const int prim, float3 *dPdu, float3 *dPdv)`
- **功能**: 计算位置关于重心坐标 u, v 的偏微分。`dPdu = p1 - p0`, `dPdv = p2 - p0`。

### triangle_attribute_dfdx() / triangle_attribute_dfdy()
- **签名**: `template<typename T> ccl_device_inline T triangle_attribute_dfdx(const differential &du, const differential &dv, const T &f0, const T &f1, const T &f2)`
- **功能**: 计算三角形属性关于屏幕空间 x/y 方向的偏微分。基于链式法则: `df/dx = (f1-f0)*du.dx + (f2-f0)*dv.dx`。

### triangle_attribute\<T\>()
- **签名**: `template<typename T> ccl_device dual<T> triangle_attribute(KernelGlobals kg, const ShaderData *sd, const AttributeDescriptor desc, const bool dx = false, const bool dy = false)`
- **功能**: 读取三角形元素上的属性值并计算偏微分。支持以下元素类型:
  - `ATTR_ELEMENT_VERTEX / VERTEX_MOTION`: 从顶点数组按顶点索引读取，重心坐标插值
  - `ATTR_ELEMENT_CORNER`: 从面角(corner)数组按 prim*3 读取
  - `ATTR_ELEMENT_CORNER_BYTE`: 字节颜色版本，通过 `attribute_data_fetch_bytecolor()` 转换
  - `ATTR_ELEMENT_FACE`: 每面属性，直接按图元索引读取

## 依赖关系
- **内部头文件**: `kernel/globals.h`, `kernel/geom/attribute.h`, `kernel/geom/object.h`
- **被引用**: `geom/primitive.h`, `geom/triangle_intersect.h`, `geom/motion_triangle.h`, `svm/wireframe.h`, `svm/bevel.h`, `osl/services_gpu.h`, `osl/services_shared.h`, `light/triangle.h`, `integrator/shade_surface.h`, `integrator/mnee.h`

## 实现细节 / 关键算法

### 属性插值
三角形属性插值使用标准重心坐标: `value = u * f1 + v * f2 + (1 - u - v) * f0`。注意 Cycles 中 f0 对应权重 w = 1-u-v (第一个顶点)，f1 对应 u，f2 对应 v。

### 偏微分计算
属性的屏幕空间微分通过链式法则从参数空间微分导出：
- `df/dx = df/du * du/dx + df/dv * dv/dx = (f1 - f0) * du.dx + (f2 - f0) * dv.dx`
- 受 `__RAY_DIFFERENTIALS__` 预处理宏保护

### 负缩放处理
`object_negative_scale_applied()` 检查对象是否应用了负缩放（镜像变换）。负缩放会翻转三角形的绕序(winding order)，因此需要反转叉积以保持法线朝向正确。

## 关联文件
- `kernel/geom/attribute.h` - 属性查找和数据获取
- `kernel/geom/object.h` - 对象变换和负缩放检查
- `kernel/geom/triangle_intersect.h` - 三角形光线相交和着色设置
- `kernel/geom/motion_triangle.h` - 运动三角形顶点和法线
- `kernel/geom/primitive.h` - 统一图元属性接口
