# primitive.h - 图元属性统一查询接口

## 概述
本文件提供了 Cycles 渲染器中所有图元类型(三角形网格、曲线、点云、体积)的属性统一查询接口。将表面属性和体积属性分开处理以优化 GPU 性能(避免引入重量级的体积插值代码)。同时提供 UV 坐标、PTEX 坐标、表面切线和运动向量(motion vector)等渲染通道的计算功能。

## 核心函数

### primitive_surface_attribute\<T\>()
- **签名**: `template<typename T> ccl_device_forceinline dual<T> primitive_surface_attribute(KernelGlobals kg, const ShaderData *sd, const AttributeDescriptor desc, const bool dx = false, const bool dy = false)`
- **功能**: 统一的表面属性读取接口。根据图元类型分发到具体实现：
  - `ATTR_ELEMENT_OBJECT / ATTR_ELEMENT_MESH`: 直接读取对象/网格级别属性
  - `PRIMITIVE_TRIANGLE`: 调用 `triangle_attribute<T>()`
  - `PRIMITIVE_CURVE`: 调用 `curve_attribute<T>()`
  - `PRIMITIVE_POINT`: 调用 `point_attribute<T>()`

### primitive_volume_attribute\<T\>()
- **签名**: `template<typename T> ccl_device_inline T primitive_volume_attribute(KernelGlobals kg, ShaderData *sd, const AttributeDescriptor desc, const bool stochastic)`
- **功能**: 体积属性读取接口。检查图元是否为体积类型，是则调用 `volume_attribute_float4()` 读取并通过 `volume_attribute_value<T>()` 转换类型。受 `__VOLUME__` 保护。

### primitive_is_volume_attribute()
- **签名**: `ccl_device_forceinline bool primitive_is_volume_attribute(const ShaderData *sd)`
- **功能**: 检查当前着色数据是否对应体积属性(类型为 `PRIMITIVE_VOLUME`)。

### primitive_uv()
- **签名**: `ccl_device_forceinline float3 primitive_uv(KernelGlobals kg, const ShaderData *sd)`
- **功能**: 获取默认 UV 坐标(`ATTR_STD_UV`)。读取 float2 属性后转换为 float3(z=1.0)。

### primitive_ptex()
- **签名**: `ccl_device bool primitive_ptex(KernelGlobals kg, ShaderData *sd, float2 *uv, int *face_id)`
- **功能**: 获取 PTEX 纹理坐标和面 ID。通过属性系统读取 `ATTR_STD_PTEX_FACE_ID` 和 `ATTR_STD_PTEX_UV`。

### primitive_tangent()
- **签名**: `ccl_device float3 primitive_tangent(KernelGlobals kg, ShaderData *sd)`
- **功能**: 获取表面切线方向。优先级：
  1. 曲线/点云: 使用 dPdu 归一化方向
  2. 三角形: 尝试从生成坐标(generated coordinates)计算球形切线
  3. 回退: 使用 dPdu

### primitive_motion_vector()
- **签名**: `ccl_device_forceinline float4 primitive_motion_vector(KernelGlobals kg, const ShaderData *sd)`
- **功能**: 计算运动向量(motion vector)渲染通道数据。处理流程：
  1. 获取中心帧位置
  2. 读取前后帧的变形运动属性
  3. 应用对象运动变换
  4. 根据相机类型(透视/正交/全景/自定义)投影到屏幕空间
  5. 返回 float4(motion_pre.xy, motion_post.xy)

## 依赖关系
- **内部头文件**: `kernel/globals.h`, `kernel/camera/projection.h`, `kernel/geom/attribute.h`, `kernel/geom/curve.h`, `kernel/geom/object.h`, `kernel/geom/point.h`, `kernel/geom/triangle.h`, `kernel/geom/volume.h`
- **被引用**: `svm/attribute.h`, `svm/vertex_color.h`, `svm/tex_coord.h`, `svm/geometry.h`, `svm/bump.h`, `svm/closure.h`, `svm/displace.h`, `osl/services_gpu.h`, `osl/services.cpp`, `osl/closures.cpp`, `film/data_passes.h`

## 实现细节 / 关键算法

### 表面/体积属性分离
将表面属性和体积属性分为两个独立的模板函数。这是关键的性能优化，尤其在 GPU 上，避免不需要体积功能的着色器引入 3D 纹理插值等重型代码。

### 运动向量计算
运动向量的计算涉及多个变换步骤：
1. **变形运动(Deformation motion)**: 通过 `ATTR_STD_MOTION_VERTEX_POSITION` 属性获取前后帧的顶点位移
2. **对象运动(Object motion)**: 通过 `object_fetch_motion_pass_transform()` 获取前后帧的对象变换
3. **相机投影**: 根据相机类型选择不同投影方式
   - 透视/正交: 使用 `worldtoraster` 矩阵
   - 全景(Panorama): 先变换到相机空间再映射到全景坐标
   - 自定义: 回退到相机空间方向向量

最终输出前后帧位置相对于中心帧的屏幕空间偏移。

## 关联文件
- `kernel/geom/attribute.h` - 属性查找基础设施
- `kernel/geom/triangle.h` - 三角形属性读取
- `kernel/geom/curve.h` - 曲线属性读取
- `kernel/geom/point.h` - 点云属性读取
- `kernel/geom/volume.h` - 体积属性读取
- `kernel/camera/projection.h` - 相机投影用于运动向量
