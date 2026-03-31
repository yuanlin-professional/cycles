# services_gpu.h - 开放着色语言(OSL)GPU 端渲染服务实现

## 概述

`services_gpu.h` 是 Cycles 渲染器中开放着色语言(OSL)在 GPU (主要是 OptiX) 上运行时的渲染服务实现。与 CPU 端的 `services.cpp` 对应,该文件提供了闭包内存管理(乘法、加法、分配)、矩阵变换查询、属性获取和纹理查找等功能的 GPU 实现。由于 GPU 上无法使用 OSL 的 ShadingSystem 对象,所有功能均以 `ccl_device` 函数形式直接实现,并通过 OptiX 的可调用程序机制使用。

## 类与结构体

本文件无独立类或结构体定义。在 `DeviceStrings` 命名空间中定义了所有 GPU 端使用的字符串常量(以哈希值形式),包括:

### `DeviceStrings` 命名空间
以 `ccl_device_constant DeviceString` 形式定义的字符串哈希常量,涵盖:
- 坐标空间: `u_common`, `u_world`, `u_shader`, `u_object`, `u_ndc`, `u_screen`, `u_camera`, `u_raster`
- 色彩系统: `u_colorsystem`
- 对象属性: `u_object_location`, `u_object_color`, `u_object_alpha`, `u_object_index`, `u_object_is_light`
- 几何属性: `u_geom_*` 系列
- 粒子属性: `u_particle_*` 系列
- 路径属性: `u_path_*` 系列
- 相机属性: `u_sensor_size`, `u_image_resolution`, `u_aperture_*`, `u_focal_distance`
- 其他: `u_bump_map_normal`, `u_normal_map_normal` 等

每个常量以 64 位无符号整型哈希值表示(例如 `u_world = 16436542438370751598ull`),与 CPU 端 `ustring` 的哈希值一一对应。

## 核心函数

### 闭包内存管理

#### `osl_mul_closure_color(ShaderGlobals *sg, OSLClosure *a, const float3 *weight)`
- **功能**: 将闭包与颜色权重相乘。从 `sg->closure_pool` 分配 `OSLClosureMul` 结构体,设置权重和子闭包。
- **优化**: 若权重为零或闭包为空返回 nullptr;若权重为 (1,1,1) 直接返回原闭包。

#### `osl_mul_closure_float(ShaderGlobals *sg, OSLClosure *a, float weight)`
- **功能**: 将闭包与标量权重相乘。逻辑与颜色版本类似。

#### `osl_add_closure_closure(ShaderGlobals *sg, OSLClosure *a, OSLClosure *b)`
- **功能**: 将两个闭包相加。从闭包池分配 `OSLClosureAdd` 结构体。
- **优化**: 若任一闭包为空,直接返回另一个。

#### `osl_allocate_closure_component(ShaderGlobals *sg, int id, int size)`
- **功能**: 分配闭包组件,权重默认为 (1,1,1)。

#### `osl_allocate_weighted_closure_component(ShaderGlobals *sg, int id, int size, const float3 *weight)`
- **功能**: 分配带权重的闭包组件。

### 工具函数

#### `osl_error` / `osl_printf` / `osl_warning` / `osl_fprintf`
- **功能**: GPU 端的日志/错误输出函数。当前实现为空操作(no-op)。

#### `osl_range_check_err(indexvalue, length, ...)`
- **功能**: 数组索引范围检查。将越界索引钳制到合法范围 [0, length-1]。

### 矩阵变换

#### `osl_get_matrix` / `osl_get_inverse_matrix` 系列
提供以下变体:
- `osl_get_matrix(sg, res, TransformationPtr)`: 获取对象到世界的变换矩阵
- `osl_get_inverse_matrix(sg, res, TransformationPtr)`: 获取世界到对象的逆变换矩阵
- `osl_get_matrix(sg, res, DeviceString from)`: 获取命名坐标空间的矩阵(支持 common, world, shader, object, NDC, screen, camera, raster)
- `osl_get_inverse_matrix(sg, res, DeviceString to)`: 获取到命名坐标空间的逆矩阵

#### `copy_matrix(float *res, const Transform &tfm)` / `copy_matrix(float *res, const ProjectionTransform &tfm)`
- **功能**: 将 Cycles 的变换矩阵转换为列优先的 4x4 浮点数组。

### 属性查询

#### `osl_get_attribute(sg, derivatives, object_name, name, type, val)`
- **功能**: GPU 端的属性查询入口。依次检查:
  1. 内置几何属性(bump_map_normal, normal_map_normal, trianglevertices 等)
  2. 对象标准属性(location, color, index, random 等)
  3. 粒子属性
  4. 曲线/毛发属性
  5. 点云属性
  6. 自定义用户属性

#### `set_attribute` 模板函数系列
- `set_attribute<float3>`: 将 float3 类型数据写入属性输出缓冲区(支持 VECTOR、NORMAL、POINT、COLOR 类型)
- `set_attribute<float>`: 将 float 类型数据写入属性输出缓冲区

#### 辅助数据设置函数:
- `set_data_float(dual1, derivatives, val)` - 写入标量及其偏导数
- `set_data_float3(dual3, derivatives, val)` - 写入三维向量及其偏导数
- `set_data_float4(dual4, derivatives, val)` - 写入四维向量及其偏导数

### 纹理查找

#### `osl_texture(sg, filename, handle, options, s, t, ..., result, ...)`
- **功能**: GPU 端 2D 纹理查找。根据纹理句柄类型分发:
  - `OSL_TEXTURE_HANDLE_TYPE_SVM`: 通过 SVM 纹理槽查找
  - `OSL_TEXTURE_HANDLE_TYPE_IES`: IES 光度曲线查找

#### `osl_texture3d(sg, filename, handle, options, P, ..., result, ...)`
- **功能**: GPU 端 3D 纹理查找,使用 `kernel_tex_image_interp_3d` 进行三线性插值。

#### `osl_environment(sg, filename, handle, options, R, ..., result, ...)`
- **功能**: GPU 端环境贴图查找。当前实现为空(返回 false)。

#### `osl_get_texture_info(sg, filename, handle, ..., dataname, datatype, data, ...)`
- **功能**: GPU 端纹理信息查询。当前实现为空(返回 false)。

## 依赖关系

- **内部头文件**:
  - `kernel/camera/camera.h` - 相机功能
  - `kernel/geom/attribute.h`, `kernel/geom/curve.h`, `kernel/geom/motion_triangle.h`, `kernel/geom/object.h`, `kernel/geom/point.h`, `kernel/geom/primitive.h`, `kernel/geom/triangle.h` - 几何子系统
  - `kernel/util/differential.h` - 微分工具
  - `kernel/util/ies.h` - IES 光度文件解析
  - `kernel/util/texture_3d.h` - 3D 纹理插值
  - `util/hash.h`, `util/transform.h` - 哈希和变换工具
  - `kernel/osl/osl.h` - OSL 着色器引擎
  - `kernel/osl/services_shared.h` - CPU/GPU 共享函数
- **被引用**:
  - `src/kernel/osl/services_optix.cu` - OptiX 入口文件包含此头文件

## 实现细节 / 关键算法

### 闭包池内存管理

GPU 端的闭包内存从 `ShaderGlobals::closure_pool`(固定 1024 字节的栈上缓冲区)分配。每次分配时:
1. 将指针对齐到目标结构体的对齐要求
2. 移动指针到已分配区域之后
3. 初始化分配的结构体

这种 bump allocator 模式避免了动态内存分配,适合 GPU 的执行模型。

### DeviceString 哈希值

GPU 上不支持完整的字符串操作,因此所有字符串比较通过预计算的 64 位哈希值完成。每个哈希值在编译时确定,与 CPU 端 `ustring` 的哈希值严格一致。这些值硬编码在源文件中,确保 CPU 和 GPU 路径行为一致。

### 纹理句柄类型编码

GPU 端纹理句柄通过位域编码类型和槽位:
- 高 2 位(第 30-31 位): 纹理类型标识
  - `0x1`: SVM 纹理
  - `0x2`: IES 光度文件
  - `0x3`: AO 或倒角
- 低 30 位: 纹理槽位索引

通过 `OSL_TEXTURE_HANDLE_TYPE(handle)` 和 `OSL_TEXTURE_HANDLE_SLOT(handle)` 宏提取类型和槽位。

### 与 CPU 版本的差异

- GPU 版本不支持运动模糊矩阵插值
- GPU 版本不支持 `trace()` / `getmessage()` 功能
- GPU 版本不支持环境贴图查找
- GPU 版本不支持 OIIO 纹理系统(仅支持 SVM 和 IES 纹理)
- GPU 版本的日志/错误输出为空操作

## 关联文件

- `src/kernel/osl/services.h` + `services.cpp` - CPU 端渲染服务(功能对等的 CPU 实现)
- `src/kernel/osl/services_shared.h` - CPU/GPU 共享的辅助函数
- `src/kernel/osl/services_optix.cu` - OptiX 入口文件,包含本头文件
- `src/kernel/osl/osl.h` - OSL 着色器引擎
- `src/kernel/osl/types.h` - DeviceString 类型和纹理句柄宏定义
