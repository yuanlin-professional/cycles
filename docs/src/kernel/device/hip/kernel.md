# kernel.cpp - HIP 内核编译入口文件

## 概述

本文件是 Cycles 渲染器 HIP（AMD GPU）内核的顶层编译入口。其结构和功能与 CUDA 的 `kernel.cu` 完全对称，仅在 HIP 设备编译模式（`__HIP_DEVICE_COMPILE__` 已定义）下生效，按顺序包含 HIP 兼容层、配置、全局数据以及 GPU 通用的图像采样和内核实现。文件扩展名为 `.cpp`（而非 CUDA 的 `.cu`），因为 HIP 编译器（hipcc）处理标准 C++ 文件。

## 核心函数/宏定义

### 编译保护
```c
#ifdef __HIP_DEVICE_COMPILE__
```
整个文件内容包裹在 HIP 设备编译宏检查中，确保仅在 AMD HIP 编译器的设备编译阶段生效（对应 CUDA 的 `__CUDA_ARCH__`）。

### 头文件包含顺序
1. **`kernel/device/hip/compat.h`** — HIP 兼容层：设备函数限定符、类型定义、数学函数
2. **`kernel/device/hip/config.h`** — 启动参数配置：线程块大小、寄存器限制
3. **`kernel/device/hip/globals.h`** — 全局数据结构：`KernelParamsHIP`、`__constant__` 变量
4. **`kernel/device/gpu/image.h`** — GPU 通用纹理图像采样
5. **`kernel/device/gpu/kernel.h`** — GPU 通用内核实现

## 依赖关系

- **内部头文件**:
  - `kernel/device/hip/compat.h`
  - `kernel/device/hip/config.h`
  - `kernel/device/hip/globals.h`
  - `kernel/device/gpu/image.h` (GPU 通用纹理采样)
  - `kernel/device/gpu/kernel.h` (GPU 通用内核函数)
- **被引用**: 由 HIP 编译工具链（hipcc）编译为 GPU 二进制，被 `src/device/hip/device_impl.cpp` 加载执行

## 实现细节 / 关键算法

### GPU 通用层复用
与 CUDA 版本相同，HIP 内核的核心逻辑完全来自 `kernel/device/gpu/image.h` 和 `kernel/device/gpu/kernel.h`。HIP 特有的部分仅在兼容层（`compat.h`）和配置（`config.h`）中体现。这种设计确保 GPU 内核逻辑的一致性，减少跨平台维护成本。

### 文件扩展名
HIP 使用 `.cpp` 扩展名而非 `.cu`，因为 HIP 编译器（hipcc）基于 Clang，可以直接编译标准 C++ 源文件，通过 `__HIP_DEVICE_COMPILE__` 宏区分主机和设备编译阶段。

### CUDA 与 HIP 入口文件对比
| 特性 | CUDA kernel.cu | HIP kernel.cpp |
|------|---------------|----------------|
| 编译保护宏 | `__CUDA_ARCH__` | `__HIP_DEVICE_COMPILE__` |
| 编译器 | nvcc | hipcc (Clang) |
| 文件扩展名 | `.cu` | `.cpp` |
| 兼容层 | `cuda/compat.h` | `hip/compat.h` |
| 配置 | `cuda/config.h` | `hip/config.h` |
| 全局数据 | `cuda/globals.h` | `hip/globals.h` |
| 共享内核 | `gpu/kernel.h` | `gpu/kernel.h`（相同） |

## 关联文件

- `src/kernel/device/cuda/kernel.cu` — CUDA 内核入口（结构完全对称）
- `src/kernel/device/hiprt/kernel.cpp` — HIPRT（HIP 光线追踪）内核入口
- `src/kernel/device/gpu/kernel.h` — GPU 通用内核实现
- `src/kernel/device/gpu/image.h` — GPU 通用纹理采样
- `src/device/hip/device_impl.cpp` — HIP 设备管理层，负责加载和启动内核
