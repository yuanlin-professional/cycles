# services_shared.h - 开放着色语言(OSL)CPU/GPU 共享渲染服务函数

## 概述

`services_shared.h` 包含在 CPU 和 GPU 端开放着色语言(OSL)渲染服务中共享的辅助函数。这些函数主要用于属性值的数据格式化输出和凹凸贴图法线的计算。该文件通过 `ccl_device` 限定符确保函数在两个平台上均可使用,避免了代码重复。

## 类与结构体

本文件无独立类或结构体定义。

## 核心函数

### `set_data_float(const dual1 data, bool derivatives, ccl_private void *val)`
- **功能**: 将标量 dual 数据(值 + 偏导数)写入输出缓冲区。
- **格式**: `fval[0] = val`, `fval[1] = dx`, `fval[2] = dy`(仅当 `derivatives == true` 时写入偏导数)

### `set_data_float3(const dual3 data, bool derivatives, ccl_private void *val)`
- **功能**: 将三维向量 dual 数据写入输出缓冲区。
- **格式**: `fval[0..2] = val`, `fval[3..5] = dx`, `fval[6..8] = dy`

### `set_data_float4(const dual4 data, bool derivatives, ccl_private void *val)`
- **功能**: 将四维向量 dual 数据写入输出缓冲区。
- **格式**: `fval[0..3] = val`, `fval[4..7] = dx`, `fval[8..11] = dy`

### `set_attribute<T>` (模板前向声明)
```cpp
template<typename T>
ccl_device_inline bool set_attribute(const dual<T> v,
                                     const TypeDesc type,
                                     bool derivatives,
                                     ccl_private void *val);
```
- **功能**: 泛型属性值设置函数的前向声明。根据 TypeDesc 类型(VECTOR、NORMAL、POINT、COLOR、FLOAT 等)将 dual 值转换为正确的输出格式。具体特化在 `services.cpp` 和 `services_gpu.h` 中实现。

### `attribute_bump_map_normal(KernelGlobals kg, const ShaderData *sd, dual3 &f)`
```cpp
ccl_device bool attribute_bump_map_normal(
    KernelGlobals kg,
    ccl_private const ShaderData *sd,
    ccl_private dual3 &f)
```
- **功能**: 计算用于凹凸贴图的光滑法线及其偏导数。该函数是 `geom:bump_map_normal` 属性查询的后端实现。
- **实现逻辑**:
  1. 仅支持三角形图元且表面有光滑法线(`SHADER_SMOOTH_NORMAL`)
  2. 根据图元类型(静态三角形或运动三角形)调用 `triangle_smooth_normal` 或 `motion_triangle_smooth_normal`
  3. 计算光滑法线值及其对 UV 的偏导数
  4. 若已应用对象变换,逆变换回局部空间
  5. 处理背面情况(翻转法线方向)
  6. 将偏导数转换为相对于法线值的增量形式(`dx -= val`, `dy -= val`)
- **返回值**: `true` 表示成功计算,`false` 表示不支持当前图元类型

## 依赖关系

- **内部头文件**:
  - `kernel/geom/motion_triangle.h` - 运动三角形法线计算
  - `kernel/geom/triangle.h` - 静态三角形法线计算
- **被引用**:
  - `src/kernel/osl/services_gpu.h` - GPU 端渲染服务中使用
  - `src/kernel/osl/services.cpp` - CPU 端渲染服务中使用

## 实现细节 / 关键算法

### 凹凸贴图法线算法

`attribute_bump_map_normal` 是实现凹凸贴图(Bump Mapping)的关键函数。OSL 的凹凸贴图需要知道着色法线在表面上的变化率(偏导数),以便计算扰动后的法线方向。

算法步骤:
1. **基准法线**: 使用几何法线(经背面处理)作为光滑法线为零时的后备值
2. **逆变换**: 将几何法线从世界空间变换到对象空间
3. **光滑法线插值**: 对静态或运动三角形分别插值顶点法线,同时计算 u/v 方向的偏导数
4. **空间转换**: 若对象变换已应用,将结果逆变换回局部空间
5. **偏导数归一化**: 将绝对偏导数转换为相对增量(`dx -= val`),使得 `val + dx` 表示 x 方向偏移后的法线

### TypeDesc 兼容

在 OptiX GPU 端,`TypeDesc` 被重定义为 `long long` 类型(通过 `#ifdef __KERNEL_OPTIX__`),因为 GPU 上无法直接使用 OIIO 的完整 `TypeDesc` 类。该整型编码与 OIIO 的 TypeDesc 内部表示兼容。

## 关联文件

- `src/kernel/osl/services.cpp` - CPU 端渲染服务的完整实现
- `src/kernel/osl/services_gpu.h` - GPU 端渲染服务的完整实现
- `src/kernel/osl/services.h` - 渲染服务类声明
- `src/kernel/geom/triangle.h` - 三角形光滑法线计算
- `src/kernel/geom/motion_triangle.h` - 运动三角形光滑法线计算
