# config.h - HIP 设备内核启动参数配置

## 概述

本文件根据 AMD HIP GPU 硬件特性定义内核启动参数，与 CUDA 的 `config.h` 功能对应。由于 AMD GPU 架构特点不同，HIP 使用统一配置（不按架构分层），线程块大小为 1024（远大于 CUDA 的 256/384），且额外定义了 HIPRT（HIP Ray Tracing）专用线程块大小。

## 核心函数/宏定义

### 硬件参数常量

| 参数 | 值 | 说明 |
|------|----|------|
| `GPU_MULTIPRESSOR_MAX_REGISTERS` | 65536 | 每个计算单元（CU）的总寄存器数 |
| `GPU_MULTIPROCESSOR_MAX_BLOCKS` | 64 | 每个 CU 的最大线程块数（CUDA 为 32） |
| `GPU_BLOCK_MAX_THREADS` | 1024 | 每个线程块的最大线程数 |
| `GPU_THREAD_MAX_REGISTERS` | 255 | 每个线程的最大寄存器数 |
| `GPU_KERNEL_BLOCK_NUM_THREADS` | 1024 | 每个线程块的实际线程数（CUDA 为 256/384） |
| `GPU_KERNEL_MAX_REGISTERS` | 64 | 每个线程的寄存器限制 |
| `GPU_HIPRT_KERNEL_BLOCK_NUM_THREADS` | 1024 | HIPRT 专用线程块大小（独立于通用设置以便调优） |

### 内核启动宏
- **`ccl_gpu_kernel(block_num_threads, thread_num_registers)`**: 与 CUDA 版本相同的 `__launch_bounds__` 定义
- **`ccl_gpu_kernel_threads(block_num_threads)`**: 简化版，仅指定线程数
- **`ccl_gpu_kernel_signature(name, ...)`**: 生成 `kernel_gpu_<name>` 函数签名
- **`ccl_gpu_kernel_postfix`**: 内核后缀（HIP 上为空）
- **`ccl_gpu_kernel_call(x)`**: 直接调用
- **`ccl_gpu_kernel_within_bounds(i, n)`**: 边界检查

### Lambda 函数对象
- **`ccl_gpu_kernel_lambda(func, ...)`**: 与 CUDA 版本相同的 `KernelLambda` 结构体定义

### 编译时健全性检查
与 CUDA 版本相同的三项静态检查，确保参数配置合理。

## 依赖关系

- **内部头文件**: 无（自包含）
- **被引用**:
  - `src/kernel/device/hip/kernel.cpp`
  - `src/kernel/device/hiprt/kernel.cpp`

## 实现细节 / 关键算法

### HIP 与 CUDA 配置差异
| 参数 | CUDA (Volta+) | HIP | 原因 |
|------|---------------|-----|------|
| 线程块大小 | 384 | 1024 | AMD CU 支持更大 wavefront 并行度 |
| 最大块数/CU | 32 | 64 | AMD 架构支持更多并发块 |
| 寄存器限制 | 168 | 64 | 不同的寄存器文件设计 |

### 无架构分层
与 CUDA 版本根据 `__CUDA_ARCH__` 分层不同，HIP 使用统一配置。这可能是因为 AMD RDNA/CDNA 架构间的差异较小，或者当前 Cycles 仅针对单一 HIP 配置优化。

### HIPRT 线程块分离
HIPRT（HIP 光线追踪）内核的线程块大小独立定义为 `GPU_HIPRT_KERNEL_BLOCK_NUM_THREADS`，当前与通用内核相同（1024），但注释说明这是为了便于独立调优性能。

## 关联文件

- `src/kernel/device/cuda/config.h` — CUDA 版本的对应配置（按架构分层，参数不同）
- `src/kernel/device/hip/compat.h` — HIP 兼容层
- `src/kernel/device/gpu/kernel.h` — 使用这里定义的启动宏
