# attribute.h - 几何属性查找与数据读取

## 概述
本文件实现了 Cycles 渲染器中几何属性(Attribute)的查找与读取机制。支持在顶点、三角形、曲线控制点、曲线、网格和体积栅格等多种网格元素上存储任意数量的属性。SVM 使用整数 ID 查找属性，而 OSL 使用 ustring 进行查找。

## 核心函数

### attribute_not_found()
- **签名**: `ccl_device_inline AttributeDescriptor attribute_not_found()`
- **功能**: 返回一个表示"属性未找到"的空描述符，element 设为 `ATTR_ELEMENT_NONE`，类型为 `ATTR_STD_NOT_FOUND`。

### object_attribute_map_offset()
- **签名**: `ccl_device_inline uint object_attribute_map_offset(KernelGlobals kg, const int object)`
- **功能**: 获取指定对象在属性映射表(attribute map)中的偏移量。

### find_attribute()
- **签名**: `ccl_device_inline AttributeDescriptor find_attribute(KernelGlobals kg, const int object, const int prim, const uint64_t id)`
- **功能**: 根据对象ID、图元ID和属性唯一ID，在属性映射表中线性搜索匹配的属性描述符。支持链式跳转(chain jump)以处理属性表分块存储的情况。如果对象为 `OBJECT_NONE` 或属性不存在则返回未找到描述符。
- **重载**: 另有接受 `ShaderData *sd` 的重载版本，从着色数据中提取 object 和 prim 参数。

### attribute_data_fetch\<T\>()
- **签名**: `template<typename T> ccl_device_inline T attribute_data_fetch(KernelGlobals kg, int offset)`
- **功能**: 模板化的属性数据读取函数，根据类型 T (float, float2, float3, float4, uchar4, Transform) 从对应的内核数据数组中获取属性值。

### attribute_data_fetch_bytecolor()
- **签名**: `template<typename T> ccl_device_inline T attribute_data_fetch_bytecolor(KernelGlobals kg, int offset)`
- **功能**: 专用于 `ATTR_ELEMENT_CORNER_BYTE` 类型属性的读取，将 uchar4 颜色值转换为线性 float4 颜色空间(sRGB 到线性转换后执行 Rec.709 到 RGB 转换)。仅支持 float4 类型特化。

### primitive_attribute_matrix()
- **签名**: `ccl_device Transform primitive_attribute_matrix(KernelGlobals kg, const AttributeDescriptor desc)`
- **功能**: 读取网格上的变换矩阵属性，从连续三个 float4 数据构造 Transform 矩阵。

## 依赖关系
- **内部头文件**: `kernel/globals.h`, `kernel/types.h`, `kernel/util/colorspace.h`, `util/color.h`
- **被引用**: `geom/curve.h`, `geom/point.h`, `geom/triangle.h`, `geom/volume.h`, `geom/primitive.h`, `svm/attribute.h`, `svm/vertex_color.h`, `svm/bump.h`, `svm/displace.h`, `osl/services_gpu.h`, `osl/closures.cpp`, `integrator/volume_shader.h`

## 实现细节 / 关键算法
- **属性查找机制**: 属性映射表以 `AttributeMap` 数组形式存储。查找时从对象的 `attribute_map_offset` 开始，逐项比对 `attr_map.id`。遇到 `ATTR_STD_NONE` 时，若 element 为 0 表示搜索结束(未找到)，否则通过 `attr_map.offset` 执行链式跳转到表的另一部分继续搜索。每次常规跳转步长为 `ATTR_PRIM_TYPES`。
- **类型系统**: 属性按元素类型(vertex, face, corner, object, mesh, voxel 等)和数据类型(float, float2, float3, float4 等)组织。不同元素类型的数据存储在不同的内核数据数组中。
- **字节颜色转换**: `ATTR_ELEMENT_CORNER_BYTE` 属性存储为 uchar4 以节省内存，读取时需要执行 sRGB 到线性颜色空间的转换。

## 关联文件
- `kernel/geom/triangle.h` - 三角形属性插值
- `kernel/geom/curve.h` - 曲线属性插值
- `kernel/geom/point.h` - 点云属性插值
- `kernel/geom/volume.h` - 体积属性读取
- `kernel/geom/primitive.h` - 图元属性统一接口
