# context_intersect_begin.h - oneAPI Embree GPU 求交上下文开始

## 概述

本文件在启用 Embree GPU 硬件光线追踪时，重定义 `ccl_gpu_kernel_signature` 宏，使包含求交操作的内核能够接收 `sycl::kernel_handler` 参数。该参数用于在运行时查询 SYCL 特化常量（specialization constants），从而动态控制 Embree 的特性标志。

本文件与 `context_intersect_end.h` 配对使用，包围包含 Embree 求交逻辑的内核代码段。

## 核心函数/宏定义

### 条件编译

仅在以下条件同时满足时生效：
- 未定义 `WITH_ONEAPI_SYCL_HOST_TASK`（非主机调试模式）
- 定义了 `WITH_EMBREE_GPU`（启用 Embree GPU 支持）

### 宏重定义

```cpp
#undef ccl_gpu_kernel_signature
#define ccl_gpu_kernel_signature(name, ...) ...
```

新的宏生成的内核入口函数与标准版本的关键区别：
- `parallel_for` 的 lambda 参数从 `sycl::nd_item<1> item` 变为 `sycl::nd_item<1> item, sycl::kernel_handler oneapi_kernel_handler`
- lambda 内部将 `oneapi_kernel_handler` 存储到 `((ONEAPIKernelContext*)kg)->kernel_handler`

这使得内核代码可以通过 `kernel_handler.get_specialization_constant<>()` 在运行时读取 Embree 特性标志。

## 依赖关系

- **内部头文件**: 无额外包含
- **被引用**:
  - `kernel/device/gpu/kernel.h` — 在包含 Embree 求交相关内核（如 `integrator_intersect_closest` 等）之前包含本文件

## 实现细节 / 关键算法

### SYCL 特化常量机制

SYCL 特化常量允许在内核编译后、执行前设置编译期常量值。Embree GPU 的特性标志（如是否支持曲线、点云、运动模糊等）通过此机制传递：

1. 主机端通过 `cgh.set_specialization_constant<ONEAPIKernelContext::oneapi_embree_features>(flags)` 设置特性值
2. `parallel_for` 的 lambda 接收 `sycl::kernel_handler`
3. 内核代码通过 `kernel_handler.get_specialization_constant<...>()` 读取值
4. SYCL 编译器根据常量值优化生成的代码

### 包围模式

本文件重定义 `ccl_gpu_kernel_signature` 后，在 `context_intersect_end.h` 中恢复为标准定义，形成一个"包围"模式。仅被此对文件包围的内核会使用 Embree GPU 的 `kernel_handler`。

## 关联文件

- `kernel/device/oneapi/context_intersect_end.h` — 配对文件，恢复标准内核签名宏
- `kernel/device/oneapi/compat.h` — 标准 `ccl_gpu_kernel_signature` 定义
- `kernel/device/oneapi/kernel.cpp` — Embree 特性标志映射函数
- `kernel/device/gpu/kernel.h` — GPU 通用内核框架
