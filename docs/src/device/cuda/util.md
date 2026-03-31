# util.h / util.cpp - CUDA 上下文管理与工具函数

## 概述

本文件提供了 CUDA 后端的基础工具设施，包括 CUDA 上下文作用域管理器 `CUDAContextScope`、错误检查宏 `cuda_assert` / `cuda_device_assert`、设备支持检测函数 `cudaSupportsDevice()`，以及非动态加载模式下的 CUEW 兼容函数。这些工具被 CUDA 后端的所有其他文件广泛使用，是整个 CUDA 子系统的基础设施层。

## 类与结构体

### `CUDAContextScope`
- **功能**: RAII 风格的 CUDA 上下文作用域管理器，在构造时将设备的 CUDA 上下文推入当前线程的上下文栈，在析构时弹出
- **关键成员**:
  - `device` (`CUDADevice*`, private) — 关联的 CUDA 设备
- **关键方法**:
  - 构造函数 — 调用 `cuCtxPushCurrent(device->cuContext)` 将上下文压入栈
  - 析构函数 — 调用 `cuCtxPopCurrent(nullptr)` 弹出上下文

## 核心函数

### 宏定义

#### `cuda_device_assert(cuda_device, stmt)`
- **功能**: 执行 CUDA API 调用 `stmt`，若返回值不为 `CUDA_SUCCESS`，则通过 `cuda_device->set_error()` 报告错误，包含错误名称、语句文本、文件名和行号
- **用法**: `cuda_device_assert(device, cuMemAlloc(&ptr, size));`

#### `cuda_assert(stmt)`
- **功能**: `cuda_device_assert` 的简化形式，将 `cuda_device` 参数固定为 `this`，用于 `CUDADevice` 类的成员函数中
- **展开**: `cuda_device_assert(this, stmt)`

### `cudaSupportsDevice(const int cudaDevID)`
- **功能**: 内联静态函数，检查指定 CUDA 设备是否被 Cycles 支持
- **判断条件**: 计算能力主版本号 >= 5（即 Maxwell 架构及以上，SM 5.0+）
- **返回值**: `bool`

### 非动态加载兼容函数（`#ifndef WITH_CUDA_DYNLOAD`）

#### `cuewErrorString(CUresult result)`
- **功能**: 将 CUDA 错误码转换为字符串表示。在非动态加载模式下（直接链接 CUDA），由于没有 CUEW 库提供此函数，这里提供一个简单的数字字符串替代实现
- **注意**: 使用静态局部变量存储结果字符串，非线程安全

#### `cuewCompilerPath()`
- **功能**: 返回 NVCC 编译器路径。在非动态加载模式下返回编译时确定的 `CYCLES_CUDA_NVCC_EXECUTABLE` 宏值

#### `cuewCompilerVersion()`
- **功能**: 返回 CUDA 编译器版本号。通过 `CUDA_VERSION` 宏计算，格式为两位数（如 `121` 表示 CUDA 12.1）

## 依赖关系

- **内部头文件**:
  - `device/cuda/device_impl.h` — 在 `.cpp` 中引用以访问 `CUDADevice::cuContext`
  - CUDA 头文件：`<cuew.h>`（动态加载模式）或 `<cuda.h>`（链接模式）
- **被引用**:
  - `src/device/cuda/device_impl.h` — `CUDADevice` 类声明中引用 `CUDAContextScope` 和错误检查宏
  - `src/device/cuda/device_impl.cpp` — 所有 CUDA API 调用使用 `cuda_assert` 宏
  - `src/device/cuda/queue.h` — `CUDADeviceQueue` 类引用 `CUDAContextScope`
  - `src/device/cuda/queue.cpp` — 队列操作使用上下文管理和错误检查
  - `src/device/cuda/graphics_interop.cpp` — 图形互操作使用上下文管理和错误检查
  - `src/device/cuda/device.cpp` — `device_cuda_info()` 使用 `cudaSupportsDevice()` 和 `cuewErrorString()`
  - `src/device/optix/util.h` — OptiX 工具层引用本文件

## 实现细节 / 关键算法

1. **上下文栈管理**: CUDA 使用线程局部的上下文栈机制。在多线程环境中，每个线程需要在执行 CUDA API 前将正确的上下文推入栈中。`CUDAContextScope` 利用 RAII 模式确保上下文在作用域结束时自动弹出，防止上下文泄漏。多 GPU 场景下，每个设备有不同的上下文，通过 `CUDAContextScope` 可以安全地在同一线程中切换不同设备的上下文。

2. **错误检查策略**: `cuda_device_assert` 宏将错误信息路由到设备对象的 `set_error()` 方法，而不是直接终止程序。这种设计允许渲染器在遇到 CUDA 错误时优雅地降级（如回退到 CPU 渲染），而非崩溃。错误消息包含完整的调用语句文本和源码位置，便于调试。

3. **动态加载抽象**: 通过条件编译，在直接链接 CUDA 库时提供与 CUEW（CUDA Extension Wrangler）兼容的函数签名，使得上层代码无需关心 CUDA 是动态加载还是静态链接。

4. **最低架构要求**: SM 5.0（Maxwell 架构）是 Cycles 支持的最低 CUDA 计算能力。低于此版本的 GPU（如 Kepler、Fermi）不再被支持。

## 关联文件

- `src/device/cuda/device_impl.h` / `device_impl.cpp` — 最主要的使用者
- `src/device/cuda/queue.h` / `queue.cpp` — 队列操作中广泛使用
- `src/device/cuda/graphics_interop.cpp` — 图形互操作使用上下文和错误检查
- `src/device/cuda/kernel.cpp` — 内核加载使用错误检查
- `src/device/optix/util.h` — OptiX 工具层基于本文件扩展
