# context_begin.h - ONEAPIKernelContext 类定义开始

## 概述

本文件定义了 `ONEAPIKernelContext` 类的开始部分。该类继承自 `KernelGlobalsGPU`，作为 oneAPI 内核的上下文容器，封装了图像采样等 GPU 通用功能。它与 `context_end.h` 配对使用，在两者之间插入 GPU 通用内核代码，使这些代码成为类的成员函数。

## 核心函数/宏定义

### ONEAPIKernelContext 类

```cpp
struct ONEAPIKernelContext : public KernelGlobalsGPU {
  public:
    #include "kernel/device/gpu/image.h"
```

**继承关系**: 继承自 `KernelGlobalsGPU`，因此拥有所有全局数据数组指针、积分器状态指针和 `KernelData` 指针等成员。

**包含的功能**: 在类体内直接包含 `kernel/device/gpu/image.h`，将 GPU 通用的图像采样函数作为成员函数引入。

### NanoVDB 支持

在类定义之前包含 `kernel/util/nanovdb.h`，提供 NanoVDB 体积数据的读取支持。

## 依赖关系

- **内部头文件**:
  - `kernel/util/nanovdb.h` — NanoVDB 体积数据支持
  - `kernel/device/gpu/image.h` — GPU 通用图像采样代码（在类内部包含）
- **被引用**:
  - `kernel/device/gpu/kernel.h` — 当 `__KERNEL_ONEAPI__` 定义时包含本文件

## 实现细节 / 关键算法

### 继承模式与 Metal 的区别

与 Metal 后端不同（Metal 使用组合模式，将 `KernelParamsMetal` 作为成员），oneAPI 使用继承模式：`ONEAPIKernelContext` 直接继承 `KernelGlobalsGPU`。这意味着：

- `KernelGlobalsGPU` 中定义的数据访问宏（如 `kernel_data_fetch`）可以直接通过 `this->` 在成员函数中使用
- 内核代码中的 `kg` 指针可以直接转型为 `ONEAPIKernelContext*`（通过 `ccl_gpu_kernel_call` 宏）

### 类拆分模式

与 Metal 的 `context_begin.h` / `context_end.h` 类似，oneAPI 也使用拆分类定义模式：
1. 本文件打开 `ONEAPIKernelContext` 类，包含图像采样功能
2. GPU 通用内核代码被包含在类体内，成为成员函数
3. `context_end.h` 关闭类定义并重定向宏

## 关联文件

- `kernel/device/oneapi/context_end.h` — 配对文件，关闭 ONEAPIKernelContext 类
- `kernel/device/oneapi/globals.h` — 定义基类 `KernelGlobalsGPU`
- `kernel/device/oneapi/compat.h` — oneAPI 兼容层
- `kernel/device/gpu/image.h` — GPU 通用图像处理代码
- `kernel/device/gpu/kernel.h` — GPU 通用内核框架
