# optimization.h - CPU 内核优化级别定义

## 概述

`optimization.h` 根据目标处理器架构定义编译期内核优化宏，控制 Cycles 为哪些指令集生成优化内核。支持 x86（32 位）、x86-64 和 ARM Neon 三种架构，通过条件编译确定可用的 SIMD 指令集级别。

## 类与结构体

无类或结构体定义。

## 核心函数/宏定义

### x86（32 位，`i386` / `_M_IX86`）

不定义任何优化宏，仅编译常规内核。

### x86-64（`__x86_64__` / `_M_X64`）

| 宏 | 说明 |
|----|------|
| `__KERNEL_SSE42__` | 自动启用——SSE4.2 是 x86-64 的最低要求 |
| `WITH_CYCLES_OPTIMIZED_KERNEL_AVX2` | 仅在 `WITH_KERNEL_AVX2` 编译选项开启时定义，编译 AVX2 优化内核 |

**注意**：x86-64 架构不单独生成 SSE4.2 内核，因为 SSE4.2 已包含在常规内核中。

### ARM Neon（`__ARM_NEON` / `_M_ARM64`，需配合 `WITH_SSE2NEON`）

| 宏 | 说明 |
|----|------|
| `__KERNEL_NEON__` | 标记使用 Neon 指令集 |
| `__KERNEL_SSE__` | 兼容 SSE 代码路径 |
| `__KERNEL_SSE2__` | 兼容 SSE2 代码路径 |
| `__KERNEL_SSE3__` | 兼容 SSE3 代码路径 |
| `__KERNEL_SSE42__` | 兼容 SSE4.2 代码路径 |

ARM Neon 通过 `sse2neon` 库模拟 SSE 指令，因此同时定义所有 SSE 级别宏以复用大部分 x86 向量化代码。

### 编译条件

所有宏仅在非 GPU 内核（`!__KERNEL_GPU__`）环境下定义。

## 依赖关系

- **内部头文件**: 无
- **外部依赖**: 无（纯预处理宏文件）
- **被引用**: `kernel/device/cpu/kernel_avx2.cpp`

## 实现细节

1. **最低要求策略**：x86-64 平台将 SSE4.2 作为最低要求，简化了内核分发逻辑——无需为 SSE2/SSE3 等旧指令集单独编译内核。

2. **ARM Neon 兼容模式**：通过定义所有 SSE 级别宏，ARM Neon 路径可以复用大量为 x86 SSE 编写的向量化代码。针对性能和兼容性差异，部分代码会检测 `__KERNEL_NEON__` 进行特殊处理。

3. **条件编译链**：`WITH_KERNEL_AVX2` 是外部构建系统（CMake）传入的编译选项，本文件将其转换为内部使用的 `WITH_CYCLES_OPTIMIZED_KERNEL_AVX2` 宏。

## 关联文件

- `util/simd.h` - 根据本文件定义的宏包含对应的 SIMD 内联函数
- `util/defines.h` - 提供基础平台检测宏（如 `__KERNEL_64_BIT__`）
- `kernel/device/cpu/kernel_avx2.cpp` - AVX2 优化内核编译单元
