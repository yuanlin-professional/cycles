# compat.h - Metal 设备兼容层与平台抽象定义

## 概述

本文件是 Cycles 渲染器 Metal 后端的核心兼容层，负责将 Cycles 的跨平台内核抽象映射到 Apple Metal Shading Language (MSL) 的具体实现。它定义了编译器标识宏、地址空间限定符、GPU 线程模型抽象、数学函数映射、MetalRT 光线追踪类型别名、纹理采样器、以及内核入口点生成宏。

本文件必须在所有其他内核头文件之前被包含，是 Metal 内核编译的基础。

## 核心函数/宏定义

### 编译器标识与命名空间

| 宏 | 说明 |
|----|------|
| `__KERNEL_GPU__` | 标识当前为 GPU 内核编译 |
| `__KERNEL_METAL__` | 标识当前为 Metal 后端 |
| `CCL_NAMESPACE_BEGIN` / `CCL_NAMESPACE_END` | 定义为空（Metal 不使用 C++ 命名空间） |

### 地址空间限定符映射

| Cycles 抽象 | Metal 映射 | 说明 |
|-------------|-----------|------|
| `ccl_global` | `device` | 设备全局内存 |
| `ccl_constant` | `constant` | 常量内存 |
| `ccl_private` | `thread` | 线程私有内存 |
| `ccl_gpu_shared` | `threadgroup` | 线程组共享内存 |
| `ccl_ray_data` | `ray_data`（启用 MetalRT 时）或 `thread` | 射线数据地址空间 |

### 函数限定符

| 宏 | 展开 |
|----|------|
| `ccl_device` | （空） |
| `ccl_device_inline` / `ccl_device_forceinline` | `__attribute__((always_inline))` |
| `ccl_device_noinline` | `__attribute__((noinline))`（非 Apple GPU 时）或空（Apple GPU） |

### GPU 线程模型

| 宏 | 说明 |
|----|------|
| `ccl_gpu_global_id_x()` | 全局线程ID（通过 `metal_global_id`） |
| `ccl_gpu_warp_size` | SIMD 组大小 |
| `ccl_gpu_syncthreads()` | 线程组屏障同步 |
| `ccl_gpu_ballot(predicate)` | SIMD 组投票操作 |

### 内核入口点生成宏

`ccl_gpu_kernel_signature(name, ...)` 是核心宏，生成：
1. 一个包含内核参数的结构体 `kernel_gpu_##name`
2. 一个 Metal `kernel` 入口函数 `cycles_metal_##name`
3. 参数结构体的 `run` 方法

存在两个版本：
- `__METAL_GLOBAL_BUILTINS__` 模式：使用全局内建变量（macOS 14+）
- 传统模式：通过入口参数属性传递线程位置信息

### 数学函数映射

将 C 标准数学函数映射到 Metal 原生函数，部分使用 `fast::` 前缀以牺牲精度换取性能：

- `sinf/cosf/tanf/expf/sqrtf/logf` → `fast::sin/cos/tan/exp/sqrt/log`
- `powf/fabsf/floorf` 等 → 对应 Metal 标准函数
- `__uint_as_float/__float_as_uint` 等 → `as_type<>` 位转换

### MetalRT 类型定义

当定义 `__METALRT__` 时，声明以下类型别名：

| 类型 | 说明 |
|------|------|
| `metalrt_as_type` | 加速结构类型（含实例化标签） |
| `metalrt_ift_type` | 求交函数表类型 |
| `metalrt_intersector_type` | 顶层求交器类型 |
| `metalrt_blas_as_type` | 底层加速结构类型 |
| `metalrt_blas_intersector_type` | 底层求交器类型 |

### 纹理与辅助结构

| 结构体 | 说明 |
|--------|------|
| `TextureParamsMetal` | 纹理参数（uint64_t 句柄） |
| `Texture2DParamsMetal` | 2D 纹理参数（含 `texture2d` 对象） |
| `MetalRTBlasWrapper` | BLAS 加速结构包装器 |
| `MetalAncillaries` | Metal 附加资源集合（纹理、加速结构、求交函数表） |

### 采样器预定义

`metal_samplers` 数组包含 8 种预定义采样器组合（最近邻/线性过滤 x 重复/钳位/零值/镜像寻址模式）。

### 向量构造函数

提供 `make_float2/3/4`、`make_int2/3/4`、`make_uint2/3/4`、`make_uchar4` 等类型构造辅助函数。

## 依赖关系

- **内部头文件**:
  - `<metal_atomic>`, `<metal_pack>`, `<metal_stdlib>`, `<simd/simd.h>` — Metal 标准库
  - `util/half.h` — 半精度浮点类型
  - `util/types.h` — Cycles 基础类型定义
- **被引用**:
  - `kernel/device/metal/kernel.metal` — Metal 内核入口文件（首先包含本文件）

## 实现细节 / 关键算法

### 参数结构体生成机制

`ccl_gpu_kernel_signature` 宏使用 `PARAMS_MAKER` 宏系列（`FN0`-`FN20`）将逗号分隔的参数列表转换为分号分隔的结构体成员声明。这使得内核参数可以通过 `params_struct->member` 的方式在 `run` 方法中被隐式访问（通过 `this->`）。

### Lambda 内核对象

`ccl_gpu_kernel_lambda` 宏生成一个 `KernelLambda` 结构体，它持有 `MetalKernelContext` 的引用，并通过 `operator()` 执行 lambda 函数体。这用于在内核中传递可调用对象。

### Apple GPU 特殊处理

Apple GPU（`__KERNEL_METAL_APPLE__`）上 `ccl_device_noinline` 被定义为空，因为 Apple 的编译器在处理 `noinline` 属性时行为不同。

## 关联文件

- `kernel/device/metal/globals.h` — Metal 全局数据结构定义
- `kernel/device/metal/function_constants.h` — Metal 函数常量定义
- `kernel/device/metal/context_begin.h` — MetalKernelContext 类开始
- `kernel/device/metal/kernel.metal` — Metal 内核主文件
- `kernel/device/gpu/kernel.h` — GPU 通用内核框架
