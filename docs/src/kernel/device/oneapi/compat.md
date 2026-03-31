# compat.h - oneAPI 设备兼容层与平台抽象定义

## 概述

本文件是 Cycles 渲染器 oneAPI (SYCL) 后端的核心兼容层，负责将 Cycles 的跨平台内核抽象映射到 Intel oneAPI/SYCL 的具体实现。它定义了编译器标识宏、函数限定符、GPU 线程模型抽象、内核入口点生成宏、数学函数映射、类型定义以及纹理采样适配器。

本文件同时支持两种执行模式：标准 SYCL `parallel_for` 设备执行和 `host_task` 主机端调试执行。

## 核心函数/宏定义

### 编译器标识

| 宏 | 说明 |
|----|------|
| `__KERNEL_GPU__` | 标识 GPU 内核编译 |
| `__KERNEL_ONEAPI__` | 标识 oneAPI 后端 |
| `__KERNEL_64_BIT__` | 标识 64 位地址空间 |
| `__KERNEL_GPU_RAYTRACING__` | 标识 GPU 硬件光线追踪支持（依赖 Embree GPU） |

### 地址空间限定符

oneAPI/SYCL 使用统一地址空间，因此大多数限定符定义为空：

| Cycles 抽象 | 展开 | 说明 |
|-------------|------|------|
| `ccl_global` | （空） | 全局内存 |
| `ccl_private` | （空） | 私有内存 |
| `ccl_constant` | `const` | 常量限定 |
| `ccl_gpu_shared` | （空） | 共享内存（SYCL 使用 local accessor） |

### 函数限定符

| 宏 | 展开 |
|----|------|
| `ccl_device` | `inline` |
| `ccl_device_inline` / `ccl_device_forceinline` | `__attribute__((always_inline))` |
| `ccl_device_noinline` | `__attribute__((noinline))` |

### 内核入口点生成

#### 标准模式（设备执行）

```cpp
ccl_gpu_kernel_signature(name, ...)
```
生成函数 `oneapi_kernel_##name`，接受 `KernelGlobalsGPU*`、全局/局部大小、`sycl::handler` 和内核参数，通过 `cgh.parallel_for(sycl::nd_range<1>(...), lambda)` 提交并行内核。

#### 主机任务模式（`WITH_ONEAPI_SYCL_HOST_TASK`）

用 `cgh.host_task()` 替代 `parallel_for`，在主机端串行模拟 GPU 执行，手动设置 `nd_item` 模拟值。用于调试目的。

### GPU 线程模型

**标准模式**（通过 `sycl::ext::oneapi::this_work_item` 访问）：

| 宏 | 说明 |
|----|------|
| `ccl_gpu_global_id_x()` | 全局线程 ID |
| `ccl_gpu_thread_idx_x` | 局部线程 ID |
| `ccl_gpu_block_dim_x` | 线程组大小 |
| `ccl_gpu_block_idx_x` | 线程组 ID |
| `ccl_gpu_warp_size` | 子组（Sub-group）大小 |
| `ccl_gpu_syncthreads()` | 线程组屏障 |
| `ccl_gpu_ballot(predicate)` | 子组投票 |

**主机任务模式**：通过 `kg->nd_item_*` 成员变量模拟以上值。

### 数学函数映射

- 标准函数：`fabsf/copysignf/asinf` 等 → `sycl::fabs/copysign/asin`
- 原生高性能函数：`sinf/cosf/powf/tanf/logf/expf/sqrtf` → `sycl::native::*`（牺牲精度换取性能）
- 位转换函数：`__uint_as_float/__float_as_uint` 等 → `sycl::bit_cast<>()`

### 纹理采样

| 函数 | 说明 |
|------|------|
| `ccl_gpu_tex_object_read_2D<T>()` | 通用模板（`static_assert(false)`，不应被调用） |
| `ccl_gpu_tex_object_read_2D<float>()` | 2D 纹理采样返回 float |
| `ccl_gpu_tex_object_read_2D<float4>()` | 2D 纹理采样返回 float4 |
| `ccl_gpu_tex_object_read_3D<T>()` | 通用 3D 模板 |
| `ccl_gpu_tex_object_read_3D<float>()` | 3D 纹理采样返回 float |
| `ccl_gpu_tex_object_read_3D<float4>()` | 3D 纹理采样返回 float4 |

纹理使用 SYCL bindless sampled images 扩展（`sycl::ext::oneapi::experimental::sampled_image_handle`），纹理句柄为 `uint64_t`。

### Lambda 内核对象

```cpp
ccl_gpu_kernel_lambda(func, ...)
```
生成 `KernelLambda` 结构体，持有 `ONEAPIKernelContext` 指针，通过 `operator()` 执行函数体。

### 调试输出

```cpp
sycl_printf(format, ...)
sycl_printf_(format)
```
封装 `sycl::ext::oneapi::experimental::printf`，使用 `CCL_ONEAPI_CONSTANT` 属性确保格式字符串在设备侧可用。

## 依赖关系

- **内部头文件**:
  - `<cstdint>`, `<math.h>` — C/C++ 标准库
  - `util/half.h` — 半精度浮点类型
  - `util/types.h` — Cycles 基础类型定义
- **被引用**:
  - `kernel/device/oneapi/kernel.cpp` — oneAPI 内核入口文件
  - `kernel/device/cpu/bvh.h` — CPU 后端的 Embree BVH 遍历（复用 oneAPI 兼容层定义）

## 实现细节 / 关键算法

### 双执行模式架构

oneAPI 后端通过 `WITH_ONEAPI_SYCL_HOST_TASK` 宏支持两种执行路径：
1. **设备模式**：使用 `cgh.parallel_for(nd_range, lambda)` 进行真正的 GPU 并行执行
2. **主机模式**：使用 `cgh.host_task(lambda)` 在 CPU 上串行循环模拟，手动为每次迭代设置线程 ID 等模拟值

主机模式主要用于开发调试，避免需要 GPU 硬件即可测试内核逻辑。

### 内核调用转发

`ccl_gpu_kernel_call(x)` 展开为 `((ONEAPIKernelContext*)kg)->x`，将内核函数调用转发到 `ONEAPIKernelContext` 对象的成员函数，与 Metal 的 `context.x` 模式类似。

### 3D 纹理支持

与 Metal 不同，oneAPI 后端额外支持 3D 纹理采样（`ccl_gpu_tex_object_read_3D`），用于 NanoVDB 体积数据的直接 GPU 采样。

## 关联文件

- `kernel/device/oneapi/globals.h` — oneAPI 全局数据结构
- `kernel/device/oneapi/context_begin.h` — ONEAPIKernelContext 类开始
- `kernel/device/oneapi/kernel.cpp` — oneAPI 内核主文件
- `kernel/device/oneapi/kernel_templates.h` — 内核调用模板
- `kernel/device/gpu/kernel.h` — GPU 通用内核框架
