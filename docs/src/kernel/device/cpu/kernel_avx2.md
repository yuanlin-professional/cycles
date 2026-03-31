# kernel_avx2.cpp - CPU AVX2 优化内核编译入口

## 概述

本文件是 Cycles CPU 内核的 AVX2 指令集优化编译入口。它设置 AVX2 及相关 SSE 系列指令集的特性宏，然后包含通用内核实现模板 `kernel_arch_impl.h`，生成以 `kernel_cpu_avx2_` 为前缀的内核函数集。当编译环境不支持 AVX2 优化时，定义 `KERNEL_STUB` 生成空函数桩。

## 核心函数/宏定义

### 指令集特性宏
当 `WITH_CYCLES_OPTIMIZED_KERNEL_AVX2` 已定义且非 32 位 GCC 时，启用以下特性：
- `__KERNEL_SSE__` — SSE 基础指令集
- `__KERNEL_SSE2__` — SSE2 指令集
- `__KERNEL_SSE3__` — SSE3 指令集
- `__KERNEL_SSSE3__` — SSSE3 指令集
- `__KERNEL_SSE42__` — SSE4.2 指令集
- `__KERNEL_AVX__` — AVX 指令集
- `__KERNEL_AVX2__` — AVX2 指令集

### KERNEL_STUB 模式
当 `WITH_CYCLES_OPTIMIZED_KERNEL_AVX2` 未定义时，定义 `KERNEL_STUB`，使所有内核函数生成断言桩，确保运行时不会误调用不支持的指令集代码。

### 架构标识
```c
#define KERNEL_ARCH cpu_avx2
```
使所有通过 `KERNEL_FUNCTION_FULL_NAME(name)` 展开的函数名带有 `cpu_avx2` 前缀。

## 依赖关系

- **内部头文件**:
  - `util/optimization.h` (优化编译标志)
  - `kernel/device/cpu/globals.h` (全局数据)
  - `kernel/device/cpu/kernel.h` (内核接口)
  - `kernel/device/cpu/kernel_arch_impl.h` (内核实现模板)
- **被引用**: 无（作为独立编译单元，编译结果通过链接使用）

## 实现细节 / 关键算法

### 32 位 GCC 限制
由于 bug #36316，在 32 位 GCC（`i386` 或 `_M_IX86`）上禁用 SSE 优化。这是因为 32 位 x86 的寄存器数量不足，SSE/AVX 指令的使用可能导致性能问题或编译器 bug。

### 编译分离策略
本文件与 `kernel.cpp` 分离编译的意义在于：
- `kernel.cpp` 使用 SSE4.2 编译标志，生成基准性能的函数
- `kernel_avx2.cpp` 使用 AVX2 编译标志（通过 CMake/编译系统设置 `-mavx2` 等），生成高性能函数
- 运行时通过 CPU 特性检测选择对应函数集

这样单个二进制文件可以同时包含两套内核实现，在不同硬件上都能正常运行并获得最佳性能。

### 注释说明
文件注释明确指出：本文件使用 AVX2 优化标志编译且几乎所有函数内联，而 `kernel.cpp` 不使用这些优化标志以兼容其他 CPU。

## 关联文件

- `src/kernel/device/cpu/kernel.cpp` — CPU 基准架构（SSE4.2）编译入口
- `src/kernel/device/cpu/kernel_arch_impl.h` — 共享的内核实现模板
- `src/kernel/device/cpu/kernel.h` — 内核函数声明
