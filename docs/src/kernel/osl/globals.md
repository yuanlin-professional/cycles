# globals.h + globals.cpp - 开放着色语言(OSL)全局数据与线程数据管理

## 概述

`globals.h` 和 `globals.cpp` 定义并实现了 Cycles 渲染器中开放着色语言(OSL)运行时所需的全局数据结构和线程私有数据管理。`OSLGlobals` 存储了整个渲染会话共享的 OSL 着色系统、纹理系统、着色器编译状态和对象名称映射;`OSLThreadData` 管理每个渲染线程独有的着色上下文和线程信息。`OSLTraceData` 为 OSL 的 `trace()` 函数调用缓存光线追踪结果。

## 类与结构体

### `OSLGlobals`
渲染会话级别的全局数据,在整个渲染过程中共享:

| 成员 | 类型 | 说明 |
|------|------|------|
| `use_shading` | `bool` | 是否启用 OSL 着色 |
| `use_camera` | `bool` | 是否启用 OSL 相机着色器 |
| `ss` | `OSL::ShadingSystem*` | OSL 着色系统指针 |
| `ts` | `OSL::TextureSystem*` | OSL 纹理系统指针 |
| `services` | `OSLRenderServices*` | 渲染服务接口指针 |
| `surface_state` | `vector<ShaderGroupRef>` | 所有表面着色器编译状态 |
| `volume_state` | `vector<ShaderGroupRef>` | 所有体积着色器编译状态 |
| `displacement_state` | `vector<ShaderGroupRef>` | 所有位移着色器编译状态 |
| `bump_state` | `vector<ShaderGroupRef>` | 所有凹凸着色器编译状态 |
| `background_state` | `ShaderGroupRef` | 背景着色器编译状态 |
| `camera_state` | `ShaderGroupRef` | 相机着色器编译状态 |
| `object_name_map` | `ObjectNameMap` | 对象名称到索引的哈希映射(键类型为 `OSLUStringHash`) |
| `object_names` | `vector<ustring>` | 对象名称列表 |

### `OSLTraceData`
OSL `trace()` 函数调用的结果缓存:

| 成员 | 类型 | 说明 |
|------|------|------|
| `ray` | `Ray` | 追踪光线 |
| `isect` | `Intersection` | 交点信息 |
| `sd` | `ShaderData` | 交点处的着色数据 |
| `setup` | `bool` | 是否已完成着色数据初始化 |
| `init` | `bool` | 是否已初始化 |
| `hit` | `bool` | 是否命中几何体 |

### `OSLThreadData`
每线程私有的 OSL 运行时数据:

| 成员 | 类型 | 说明 |
|------|------|------|
| `globals` | `OSLGlobals*` | 指向全局数据的指针 |
| `ss` | `OSL::ShadingSystem*` | 着色系统指针(从全局数据复制) |
| `thread_index` | `int` | 线程索引 |
| `shader_globals` | `ShaderGlobals` (mutable) | 当前着色点的全局变量 |
| `tracedata` | `OSLTraceData` (mutable) | trace() 调用结果缓存 |
| `osl_thread_info` | `OSL::PerThreadInfo*` | OSL 线程信息句柄 |
| `context` | `OSL::ShadingContext*` | OSL 着色上下文 |
| `oiio_thread_info` | `OIIO::TextureSystem::Perthread*` | OIIO 纹理线程信息 |

## 核心函数

### `OSLThreadData::OSLThreadData(OSLGlobals *osl_globals, int thread_index)` (构造函数)
- **功能**: 初始化线程数据。若 OSL 未启用(全局指针为空或着色和相机均未启用),则跳过初始化。否则:
  1. 从全局数据获取着色系统指针
  2. 将 `shader_globals` 清零并关联 `tracedata`
  3. 通过 `ss->create_thread_info()` 创建 OSL 线程信息
  4. 通过 `ss->get_context()` 获取着色上下文
  5. 若纹理系统存在,获取 OIIO 线程信息

### `OSLThreadData::~OSLThreadData()` (析构函数)
- **功能**: 释放着色上下文和线程信息。按正确顺序调用 `ss->release_context()` 和 `ss->destroy_thread_info()`。

### `OSLThreadData::OSLThreadData(OSLThreadData &&other)` (移动构造函数)
- **功能**: 支持线程数据的移动语义。将所有成员从源对象转移,并将源对象的指针置空。特别注意需要重新设置 `shader_globals.tracedata` 指向新对象的 `tracedata` 成员。

## 依赖关系

### globals.h
- **内部头文件**:
  - `<OSL/oslexec.h>` - OSL 执行引擎
  - `util/map.h`, `util/param.h`, `util/vector.h` - 工具库
  - `kernel/types.h` - 内核类型定义
  - `kernel/osl/compat.h` - OSL 版本兼容层
  - `kernel/osl/types.h` - OSL 类型定义
- **被引用**:
  - `src/kernel/osl/closures.cpp` - 着色器执行时访问着色器状态
  - `src/kernel/osl/services.cpp` - 渲染服务实现
  - `src/kernel/osl/globals.cpp` - 自身实现文件
  - `src/kernel/device/cpu/globals.cpp` - CPU 设备线程数据初始化
  - `src/device/cpu/device_impl.h` - CPU 设备实现中管理 OSL 线程数据

### globals.cpp
- **内部头文件**:
  - `<OSL/oslexec.h>` - OSL 执行引擎(需要 `int32_t` 的 `<cstdint>` 前置包含)
  - `kernel/osl/globals.h` - 对应头文件

## 实现细节 / 关键算法

1. **条件编译**: 整个 `globals.h` 内容受 `#ifdef WITH_OSL` 保护,仅在 OSL 支持编译时生效。

2. **线程安全设计**: `shader_globals` 和 `tracedata` 声明为 `mutable`,因为它们在 `const ThreadKernelGlobalsCPU*` 上下文中被修改(着色器执行本质上是修改线程局部状态)。

3. **移动语义**: `OSLThreadData` 禁止拷贝构造和拷贝赋值,仅支持移动构造。这确保了 OSL 线程信息和着色上下文的唯一所有权,避免双重释放。

4. **GCC 15.1 兼容性**: `globals.cpp` 中在包含 `<OSL/oslexec.h>` 前需要先包含 `<cstdint>`,以确保 `int32_t` 类型在 GCC 15.1 下可用。

5. **着色器状态索引**: `surface_state`、`volume_state`、`displacement_state`、`bump_state` 均为向量,以着色器索引(`sd->shader & SHADER_MASK`)进行访问。`background_state` 和 `camera_state` 是单一实例。

## 关联文件

- `src/kernel/osl/closures.cpp` - 使用 `OSLGlobals` 中的着色器状态执行着色器
- `src/kernel/osl/services.cpp` - 渲染服务通过全局数据访问纹理和属性
- `src/kernel/device/cpu/globals.cpp` - 创建 `OSLThreadData` 实例
- `src/device/cpu/device_impl.h` - 管理 `OSLThreadData` 的生命周期
- `src/scene/osl.cpp` - 场景层填充 `OSLGlobals` 数据
