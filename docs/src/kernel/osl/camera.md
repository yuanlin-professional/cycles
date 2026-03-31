# camera.h - 开放着色语言(OSL)相机着色器求值接口

## 概述

`camera.h` 提供了 Cycles 渲染器中开放着色语言(OSL)相机着色器的求值接口。该文件定义了将传感器数据转换为着色器全局变量的辅助函数,以及在 CPU 和 GPU (OptiX) 上执行相机着色器的核心函数 `osl_eval_camera`。相机着色器允许用户通过开放着色语言(OSL)自定义光线生成方式,从而实现非标准投影效果。

## 类与结构体

本文件无独立类或结构体定义,但依赖以下外部类型:
- **`ShaderGlobals`**: 开放着色语言(OSL)着色器全局变量结构体(定义于 `types.h`)
- **`KernelGlobals`**: 内核全局数据访问句柄

## 核心函数

### `cameradata_to_shaderglobals`
```cpp
ccl_device_inline void cameradata_to_shaderglobals(
    const packed_float3 sensor,
    const packed_float3 dSdx,
    const packed_float3 dSdy,
    const float2 rand_lens,
    ccl_private ShaderGlobals *globals)
```
- **功能**: 将相机传感器位置数据填充到 `ShaderGlobals` 结构体中。首先将结构体清零,然后设置位置 `P`、位置偏导数 `dPdx`/`dPdy`,以及将镜头随机采样值编码到法线 `N` 字段中。
- **参数**:
  - `sensor`: 传感器上的采样位置
  - `dSdx`, `dSdy`: 传感器位置对屏幕坐标的偏导数
  - `rand_lens`: 镜头上的随机采样坐标
  - `globals`: 输出的着色器全局变量

### `osl_eval_camera`
```cpp
packed_float3 osl_eval_camera(KernelGlobals kg,
    const packed_float3 sensor, const packed_float3 dSdx, const packed_float3 dSdy,
    const float2 rand_lens,
    packed_float3 &P, packed_float3 &dPdx, packed_float3 &dPdy,
    packed_float3 &D, packed_float3 &dDdx, packed_float3 &dDdy)
```
- **功能**: 执行开放着色语言(OSL)相机着色器,计算光线的起点和方向。
- **CPU 版本**: 声明为外部函数,实际实现在 `closures.cpp` 中,通过 `OSL::ShadingSystem::execute` 调用相机着色器状态。
- **GPU 版本** (OptiX): 内联实现,通过 `optixDirectCall` 调用编译后的相机着色器程序,输出存储在 21 个浮点数的数组中,依次包含 P(3)、dPdx(3)、dPdy(3)、D(3)、dDdx(3)、dDdy(3) 和返回光谱值(3)。
- **返回值**: 光谱权重(packed_float3),用于调节光线的贡献度。

## 依赖关系

- **内部头文件**:
  - `kernel/globals.h` - 内核全局变量定义
  - `kernel/osl/types.h` - 开放着色语言(OSL)类型定义(ShaderGlobals 等)
- **被引用**:
  - `src/kernel/osl/closures.cpp` - 提供 CPU 端 `osl_eval_camera` 的完整实现

## 实现细节 / 关键算法

1. **CPU/GPU 分支**: 通过 `__KERNEL_GPU__` 宏区分两种实现路径。CPU 版本仅声明函数原型,实际实现依赖 OSL 的 ShadingSystem;GPU 版本使用 OptiX 的 `optixDirectCall` 机制直接调用预编译的着色器。

2. **输出编码约定**: 相机着色器的输出统一编码为 21 个浮点数的数组:
   - `[0-2]`: 光线起点 P
   - `[3-8]`: P 的屏幕空间偏导数 dPdx, dPdy
   - `[9-11]`: 光线方向 D
   - `[12-17]`: D 的屏幕空间偏导数 dDdx, dDdy
   - `[18-20]`: 光谱返回值

3. **OptiX 可调用程序索引**: GPU 路径中,相机着色器使用固定的可调用程序索引 `2`(即 `NUM_CALLABLE_PROGRAM_GROUPS`)。

## 关联文件

- `src/kernel/osl/closures.cpp` - 包含 `osl_eval_camera` 的 CPU 实现
- `src/kernel/osl/types.h` - ShaderGlobals 结构体定义
- `src/kernel/osl/osl.h` - 开放着色语言(OSL)着色器引擎主入口
