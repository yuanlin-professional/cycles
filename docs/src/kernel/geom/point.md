# point.h - 点云图元着色数据与属性读取

## 概述
本文件实现了 Cycles 渲染器中点云(Point Cloud)图元的着色相关功能。提供了点云属性读取、点的位置和半径查询、随机数属性以及运动模糊中心位置计算。点云图元用于渲染粒子系统和点云数据。

## 核心函数

### point_attribute\<T\>()
- **签名**: `template<typename T> ccl_device dual<T> point_attribute(KernelGlobals kg, const ShaderData *sd, const AttributeDescriptor desc, const bool dx = false, const bool dy = false)`
- **功能**: 读取点云元素上的属性值。对于 `ATTR_ELEMENT_VERTEX` 类型，直接从属性数组中按图元索引读取。点云属性不支持微分计算(dx/dy 参数被忽略)。

### point_position()
- **签名**: `ccl_device float3 point_position(KernelGlobals kg, const ShaderData *sd)`
- **功能**: 获取点云图元在世界空间中的中心位置。支持运动模糊场景(通过 `motion_point()` 获取插值位置)和静态场景(直接读取 `points` 数组)。若变换尚未应用则执行对象到世界空间变换。

### point_radius()
- **签名**: `ccl_device float point_radius(KernelGlobals kg, const ShaderData *sd)`
- **功能**: 获取点云图元在世界空间中的半径。若变换已应用则直接返回存储的半径值；否则需要考虑对象变换的缩放影响，将归一化半径向量(每个分量为 r/sqrt(3))变换到世界空间后取长度。

### point_random()
- **签名**: `ccl_device float point_random(KernelGlobals kg, const ShaderData *sd)`
- **功能**: 获取点云的随机数属性(`ATTR_STD_POINT_RANDOM`)，用于着色器变化。

### point_motion_center_location()
- **签名**: `ccl_device float3 point_motion_center_location(KernelGlobals kg, const ShaderData *sd)`
- **功能**: 获取运动通道(motion pass)中点的中心位置(不含运动插值)，直接从静态 `points` 数组读取。

## 依赖关系
- **内部头文件**: `kernel/globals.h`, `kernel/geom/attribute.h`, `kernel/geom/motion_point.h`, `kernel/geom/object.h`
- **被引用**: `geom/primitive.h`, `osl/services_gpu.h`

## 实现细节 / 关键算法

### 半径变换
点的半径在对象空间中存储为标量值。变换到世界空间时，不能简单乘以缩放因子(因为可能存在非均匀缩放)。采用的方法是将归一化半径向量 `(r/sqrt(3), r/sqrt(3), r/sqrt(3))` 进行方向变换后取长度，这在均匀缩放时给出精确结果，非均匀缩放时给出合理近似。

### 属性系统
点云属性比曲线和三角形简单，不支持微分(无 du/dv 概念)，每个点只有一个属性值。

整个文件被 `__POINTCLOUD__` 预处理宏保护。

## 关联文件
- `kernel/geom/point_intersect.h` - 点云光线相交测试与着色设置
- `kernel/geom/motion_point.h` - 运动模糊点位置插值
- `kernel/geom/attribute.h` - 属性查找与读取基础设施
- `kernel/geom/primitive.h` - 统一图元属性接口
