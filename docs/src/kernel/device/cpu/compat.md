# compat.h - CPU 设备兼容性与断言定义

## 概述

本文件为 Cycles 渲染器 CPU 内核提供兼容性定义。主要功能是抑制 GCC 在 Release 构建下的误报未初始化警告，以及定义 CPU 专用的内核断言宏 `kernel_assert`。该文件是所有 CPU 内核代码的基础兼容层。

## 核心函数/宏定义

### 编译器警告抑制
```c
#pragma GCC diagnostic ignored "-Wmaybe-uninitialized"
#pragma GCC diagnostic ignored "-Wuninitialized"
```
仅在 GCC（非 Clang）且 `NDEBUG` 定义时生效，抑制 Release 构建中大量的误报未初始化变量警告，避免掩盖真正的警告信息。

### `kernel_assert(cond)`
将标准 `assert()` 封装为内核断言宏。在 CPU 设备上，断言正常工作；在 GPU 设备（CUDA、HIP 等）的对应 `compat.h` 中，该宏被定义为空操作。这使得内核代码可以统一使用 `kernel_assert`，仅在 CPU 上执行实际检查。

## 依赖关系

- **内部头文件**: 无（自包含）
- **被引用**:
  - `src/kernel/types.h`
  - `src/kernel/globals.h`
  - `src/kernel/device/cpu/bvh.h`
  - `src/kernel/device/cpu/image.h`
  - `src/kernel/device/cpu/kernel_arch_impl.h`

## 实现细节 / 关键算法

本文件非常精简，核心逻辑仅包含条件编译的警告抑制和一个宏定义。设计思路是：
- CPU 内核可以在开发和调试阶段充分利用断言进行数据校验
- Release 构建中 GCC 的未初始化变量警告由于内核代码的复杂控制流而产生大量误报，故统一抑制

## 关联文件

- `src/kernel/device/cuda/compat.h` — CUDA 设备兼容层（`kernel_assert` 为空）
- `src/kernel/device/hip/compat.h` — HIP 设备兼容层（`kernel_assert` 为空）
- `src/kernel/device/cpu/globals.h` — 依赖本文件提供的兼容环境
