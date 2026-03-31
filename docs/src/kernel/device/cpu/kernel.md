# kernel.h + kernel.cpp - CPU 内核公共接口声明与入口实现

## 概述

`kernel.h` 是 CPU 内核的公共接口头文件，声明了内核全局数据管理函数、内存拷贝函数，并通过宏展开为 `cpu` 和 `cpu_avx2` 两种架构分别声明所有内核函数原型。`kernel.cpp` 是 CPU 基准架构（SSE4.2）的编译入口，实现了内存拷贝函数并包含内核实际实现模板，同时设置 SSE 系列指令集特性宏。

## 核心函数/宏定义

### kernel.h — 接口声明

#### 函数名宏系统
- **`KERNEL_NAME_JOIN(x, y, z)`**: 将三个标识符用下划线连接
- **`KERNEL_NAME_EVAL(arch, name)`**: 生成 `kernel_<arch>_<name>` 格式的函数名
- **`KERNEL_FUNCTION_FULL_NAME(name)`**: 用当前 `KERNEL_ARCH` 展开完整函数名

#### 全局数据管理
- **`kernel_globals_create()`**: 创建并返回 `KernelGlobalsCPU` 实例
- **`kernel_globals_free(kg)`**: 释放全局数据
- **`kernel_osl_memory(kg)`**: 获取 OSL 内存指针
- **`kernel_osl_use(kg)`**: 查询是否使用 OSL

#### 内存拷贝
- **`kernel_const_copy(kg, name, host, size)`**: 拷贝常量数据到全局结构
- **`kernel_global_memory_copy(kg, name, mem, size)`**: 拷贝全局内存数组到全局结构

#### 架构函数声明
通过两次 `#include "kernel/device/cpu/kernel_arch.h"` 分别以 `KERNEL_ARCH=cpu` 和 `KERNEL_ARCH=cpu_avx2` 展开，为两种架构声明完整的内核函数原型。

### kernel.cpp — 入口实现

#### SSE 特性宏设置
在 x86-64 平台上定义 `__KERNEL_SSE__` 到 `__KERNEL_SSE42__`，启用 SSE4.2 作为 CPU 基准指令集。在 `WITH_KERNEL_NATIVE` 模式下根据编译器标志进一步启用 AVX/AVX2。

#### 内存拷贝实现
- **`kernel_const_copy()`**: 仅处理 `"data"` 名称，将 `KernelData` 拷贝到 `kg->data`
- **`kernel_global_memory_copy()`**: 通过 `#include "kernel/data_arrays.h"` 宏展开，匹配名称字符串并设置对应数组的 `data` 指针和 `width`（元素数量）

#### 内核实现包含
以 `KERNEL_ARCH=cpu` 包含 `kernel_arch_impl.h`，实例化所有 CPU 基准架构的内核函数。

## 依赖关系

- **内部头文件**:
  - `kernel/types.h` (内核类型)
  - `util/half.h` (半精度浮点)
  - `kernel/device/cpu/globals.h` (全局数据结构)
  - `kernel/device/cpu/kernel_arch.h` (架构函数声明模板)
  - `kernel/device/cpu/kernel_arch_impl.h` (架构函数实现模板)
  - `kernel/data_arrays.h` (数据数组列表，用于内存拷贝)
- **被引用**:
  - `src/kernel/device/cpu/kernel_avx2.cpp` (AVX2 编译单元)
  - `src/device/cpu/device_impl.cpp` (CPU 设备实现)
  - `src/device/cpu/device_impl.h` (CPU 设备头文件)
  - `src/device/cpu/kernel.cpp` (CPU 内核调度)

## 实现细节 / 关键算法

### 多架构编译策略
Cycles 的 CPU 内核采用多编译单元策略：
1. `kernel.cpp` 以 SSE4.2 为基准编译，生成 `kernel_cpu_*` 系列函数
2. `kernel_avx2.cpp` 以 AVX2 优化编译，生成 `kernel_cpu_avx2_*` 系列函数
3. 运行时根据 CPU 能力选择对应函数集

通过 `KERNEL_ARCH` 宏和 `KERNEL_FUNCTION_FULL_NAME` 宏系统，同一套声明/实现模板可以为不同架构生成不同名称的函数，避免符号冲突。

### 字符串匹配内存拷贝
`kernel_global_memory_copy` 使用 `strcmp` 逐一匹配数据数组名称。虽然不够高效，但该函数仅在场景数据加载时调用，不在渲染热路径上。

## 关联文件

- `src/kernel/device/cpu/kernel_arch.h` — 架构函数声明模板
- `src/kernel/device/cpu/kernel_arch_impl.h` — 架构函数实现模板
- `src/kernel/device/cpu/kernel_avx2.cpp` — AVX2 优化编译入口
- `src/device/cpu/device_impl.cpp` — CPU 设备层，调用本文件声明的接口
