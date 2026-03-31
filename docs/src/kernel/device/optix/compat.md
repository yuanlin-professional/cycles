# compat.h - OptiX 设备兼容层与平台抽象定义

## 概述

本文件为 OptiX 后端提供平台兼容层，将 Cycles 的跨平台抽象宏映射到 CUDA/OptiX 的具体实现。它定义了设备函数修饰符、内存空间限定符、基本类型以及纹理操作的 CUDA 实现，使得 Cycles 内核代码可以在 OptiX 平台上无缝编译。所有 OptiX 内核（包括 `kernel.cu`、`kernel_osl.cu` 等）都以本文件作为最先包含的头文件。

## 核心函数/宏定义

### 编译器预定义

- **`__KERNEL_GPU__`** - 标记 GPU 内核编译。
- **`__KERNEL_CUDA__`** - OptiX 内核隐式也是 CUDA 内核。
- **`__KERNEL_OPTIX__`** - 标记 OptiX 后端编译。
- **`CCL_NAMESPACE_BEGIN` / `CCL_NAMESPACE_END`** - 定义为空，GPU 代码不使用命名空间。

### 设备函数修饰符

- **`ccl_device`** - `static __device__ __forceinline__`，OptiX 性能对函数调用非常敏感，因此所有设备函数默认强制内联。
- **`ccl_device_extern`** - `extern "C" __device__`，用于外部可见的设备函数。
- **`ccl_device_noinline`** - `static __device__ __noinline__`，禁止内联的设备函数。
- **`ccl_device_noinline_cpu`** - 在 GPU 上等同于 `ccl_device`（仍然内联）。

### 内存空间限定符

- **`ccl_global`** - 空定义（CUDA 统一内存模型）。
- **`ccl_constant`** - `const`。
- **`ccl_inline_constant`** - `static __constant__`，CUDA 常量内存。
- **`ccl_device_constant`** - `__constant__ __device__`。
- **`ccl_gpu_shared`** - `__shared__`，GPU 共享内存。
- **`ccl_private`** / **`ccl_ray_data`** - 空定义，CUDA 默认寄存器/局部存储。
- **`ccl_restrict`** - `__restrict__`，非别名指针优化。
- **`ccl_align(n)`** - `__align__(n)`，内存对齐。

### 基本类型与工具

- **`half`** 类型 - `unsigned short`，16 位浮点数。
- **`__float2half(f)`** / **`__half2float(h)`** - 使用 PTX 内联汇编（`cvt.rn.f16.f32` / `cvt.f32.f16`）实现的半精度浮点转换。
- **`CUtexObject` / `ccl_gpu_tex_object_2D`** - `unsigned long long`，CUDA 纹理对象。
- **`ccl_gpu_tex_object_read_2D<T>(texobj, x, y)`** - 封装 `tex2D<T>` 的 2D 纹理读取。

### 结构体初始化

- **`ccl_optional_struct_init`** - `= {}`，零初始化结构体以帮助编译器推断作用域。

### 断言

- **`kernel_assert(cond)`** - 空定义，CUDA 不支持断言。

## 依赖关系

- **内部头文件**:
  - `util/half.h` - 半精度浮点工具
  - `util/types.h` - 基础类型定义
  - `<optix.h>` - OptiX SDK 头文件（使用 `OPTIX_DONT_INCLUDE_CUDA` 避免重复包含 CUDA 头文件）

- **被引用**:
  - `kernel/device/optix/kernel.cu` - OptiX 主内核
  - `kernel/device/optix/kernel_osl_camera.cu` - OSL 相机内核
  - `kernel/osl/services_optix.cu` - OSL 服务 OptiX 实现

## 实现细节 / 关键算法

1. **强制内联策略**: OptiX 的函数调用开销极高（涉及 continuation passing），因此 `ccl_device` 默认使用 `__forceinline__`。只有极少数函数使用 `ccl_device_noinline` 禁止内联。

2. **CUDACC_RTC 兼容**: 当使用 CUDA 运行时编译（`__CUDACC_RTC__`）时，标准库头文件不可用，因此手动定义 `uint32_t` 和 `uint64_t`。

3. **CYCLES_CUBIN_CC 浮点常量**: 当使用自定义 CUDA 编译器时，`FLT_MIN`、`FLT_MAX`、`FLT_EPSILON` 可能未定义，需手动提供。

4. **OptiX 头文件隔离**: 使用 `OPTIX_DONT_INCLUDE_CUDA` 宏阻止 `<optix.h>` 包含 CUDA 头文件，因为兼容层已经提供了所需的类型定义，避免重定义冲突。

## 关联文件

- `kernel/device/optix/globals.h` - 通常与本文件一起被包含
- `kernel/device/optix/kernel.cu` - 主编译单元
- `kernel/device/cuda/compat.h` - CUDA 后端的类似兼容层（结构相似但实现不同）
- `kernel/device/hip/compat.h` - HIP 后端的类似兼容层
- `util/half.h` - 半精度浮点实现
- `util/types.h` - 基础数学类型
