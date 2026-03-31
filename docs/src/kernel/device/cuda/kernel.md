# kernel.cu - CUDA 内核编译入口文件

## 概述

本文件是 Cycles 渲染器 CUDA 内核的顶层编译入口。它仅在 CUDA 设备编译模式（`__CUDA_ARCH__` 已定义）下生效，按顺序包含 CUDA 兼容层、配置、全局数据以及 GPU 通用的图像采样和内核实现。该文件本身不包含任何逻辑代码，仅作为头文件组装点，将所有必要组件串联为一个完整的 CUDA 内核编译单元。

## 核心函数/宏定义

### 编译保护
```c
#ifdef __CUDA_ARCH__
```
整个文件内容包裹在 CUDA 架构宏检查中，确保仅在 NVIDIA CUDA 编译器（nvcc 或 nvrtc）的设备编译阶段生效。

### 头文件包含顺序
1. **`kernel/device/cuda/compat.h`** — CUDA 兼容层：设备函数限定符、类型定义、数学函数
2. **`kernel/device/cuda/config.h`** — 启动参数配置：线程块大小、寄存器限制、`__launch_bounds__`
3. **`kernel/device/cuda/globals.h`** — 全局数据结构：`KernelParamsCUDA`、`__constant__` 变量、访问宏
4. **`kernel/device/gpu/image.h`** — GPU 通用纹理图像采样（使用硬件纹理单元）
5. **`kernel/device/gpu/kernel.h`** — GPU 通用内核实现：积分器、胶片转换、着色器评估等所有 GPU 内核函数

## 依赖关系

- **内部头文件**:
  - `kernel/device/cuda/compat.h`
  - `kernel/device/cuda/config.h`
  - `kernel/device/cuda/globals.h`
  - `kernel/device/gpu/image.h` (GPU 通用纹理采样)
  - `kernel/device/gpu/kernel.h` (GPU 通用内核函数)
- **被引用**: 由 CUDA 编译工具链（nvcc）编译为 .cubin 或 .ptx，被 `src/device/cuda/device_impl.cpp` 加载执行

## 实现细节 / 关键算法

### 编译流程
CUDA 内核的编译流程为：
1. nvcc 编译 `kernel.cu`，`__CUDA_ARCH__` 被设置为目标 GPU 架构值
2. `config.h` 根据架构值选择合适的启动参数
3. `gpu/kernel.h` 中的内核函数通过 `ccl_gpu_kernel` 宏获得 `__launch_bounds__` 属性
4. 编译器根据寄存器限制优化代码生成

### GPU 通用层复用
`kernel/device/gpu/image.h` 和 `kernel/device/gpu/kernel.h` 是所有 GPU 后端（CUDA、HIP、Metal、oneAPI）共享的通用实现。每个后端通过自己的 `compat.h` 提供平台特定的宏映射，通用层只使用抽象的 `ccl_*` 接口。

## 关联文件

- `src/kernel/device/hip/kernel.cpp` — HIP 内核入口（结构完全相同，使用 HIP 兼容层）
- `src/kernel/device/gpu/kernel.h` — GPU 通用内核实现（所有 GPU 内核函数的实际代码）
- `src/kernel/device/gpu/image.h` — GPU 通用纹理采样
- `src/device/cuda/device_impl.cpp` — CUDA 设备管理层，负责加载和启动内核
