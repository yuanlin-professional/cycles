# volume.h - 体积图元属性读取

## 概述
本文件实现了 Cycles 渲染器中体积(Volume)图元的属性读取功能。体积是网格内部的区域，以网格表面作为边界。与表面图元相比，体积的数据访问较简单，主要是在 3D 体素(voxel)或程序纹理中进行位置查找。3D 体素纹理可以作为网格属性分配，使同一着色器用于不同密度的体积对象。

## 核心函数

### volume_normalized_position()
- **签名**: `ccl_device_inline float3 volume_normalized_position(KernelGlobals kg, const ShaderData *sd, float3 P)`
- **功能**: 将位置归一化到网格包围盒的 [0, 1] 范围。先执行世界空间到对象空间的逆变换，再查找 `ATTR_STD_GENERATED_TRANSFORM` 属性获取生成坐标变换矩阵并应用。

### volume_attribute_value\<T\>()
- **签名**: `template<typename T> ccl_device_inline T volume_attribute_value(const float4 value)`
- **功能**: 将 float4 原始体积属性值转换为目标类型 T。各类型的转换规则：
  - `float`: 取 RGB 三通道的平均值
  - `float2`: 不支持（触发断言失败）
  - `float3`: 若 alpha > 1e-6 且 != 1.0 则执行预乘除(unpremultiply)
  - `float4`: 直接返回原值

### volume_attribute_alpha()
- **签名**: `ccl_device float volume_attribute_alpha(const float4 value)`
- **功能**: 提取体积属性值的 alpha 通道(w 分量)。

### volume_attribute_float4()
- **签名**: `ccl_device float4 volume_attribute_float4(KernelGlobals kg, ShaderData *sd, const AttributeDescriptor desc, const bool stochastic)`
- **功能**: 体积属性的底层读取函数。支持三种元素类型：
  - `ATTR_ELEMENT_OBJECT / ATTR_ELEMENT_MESH`: 从 `attributes_float4` 直接读取常量值
  - `ATTR_ELEMENT_VOXEL`: 将位置变换到对象空间后，通过 `kernel_tex_image_interp_3d()` 进行 3D 纹理采样。根据 `SD_VOLUME_CUBIC` 标志选择三次(cubic)或默认插值方式。支持随机采样(stochastic)模式。

## 依赖关系
- **内部头文件**: `kernel/globals.h`, `kernel/geom/attribute.h`, `kernel/geom/object.h`, `kernel/sample/lcg.h`, `kernel/util/texture_3d.h`
- **被引用**: `geom/primitive.h`, `svm/attribute.h`

## 实现细节 / 关键算法

### 体素纹理采样
体积属性的核心是 3D 纹理插值:
1. 将世界空间位置变换到对象空间: `object_inverse_position_transform()`
2. 调用 `kernel_tex_image_interp_3d()` 进行 3D 纹理查找
3. 插值类型由 `SD_VOLUME_CUBIC` 标志控制(三次插值提供更平滑的结果但计算更昂贵)

### RGBA 颜色处理
对于 float3 类型的体积属性值，如果 alpha 通道非零且不为 1.0，执行预乘除(unpremultiply): `rgb = rgb / alpha`。这是因为体积纹理通常以预乘 alpha 格式存储，在插值后需要恢复。

### 生成坐标变换
`volume_normalized_position()` 通过查找 `ATTR_STD_GENERATED_TRANSFORM` 属性获取一个额外的变换矩阵，将对象空间坐标映射到 [0,1] 归一化空间。此变换编码了体素纹理在对象空间中的偏移和缩放。

整个文件被 `__VOLUME__` 预处理宏保护。

## 关联文件
- `kernel/geom/attribute.h` - 属性查找和矩阵属性读取
- `kernel/geom/object.h` - 对象空间变换
- `kernel/geom/primitive.h` - 统一图元属性接口中调用体积属性
- `kernel/util/texture_3d.h` - 3D 纹理插值实现
- `kernel/sample/lcg.h` - 随机采样支持
