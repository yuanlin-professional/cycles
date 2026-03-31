# config.h - CUDA 设备内核启动参数配置

## 概述

本文件根据 CUDA GPU 架构（Compute Capability）定义内核启动参数，包括每个多处理器的最大寄存器数、最大线程块数、每个线程块的线程数以及每个线程的最大寄存器数。这些参数直接影响 GPU 占用率（occupancy），是性能调优的核心配置。同时定义了内核启动的 `__launch_bounds__` 宏和 lambda 函数对象宏。

## 核心函数/宏定义

### 硬件参数常量

#### 架构 5.x/6.x（Maxwell/Pascal，`__CUDA_ARCH__ <= 699`）
| 参数 | 值 | 说明 |
|------|----|------|
| `GPU_MULTIPRESSOR_MAX_REGISTERS` | 65536 | 每个多处理器的总寄存器数 |
| `GPU_MULTIPROCESSOR_MAX_BLOCKS` | 32 | 每个多处理器的最大线程块数 |
| `GPU_BLOCK_MAX_THREADS` | 1024 | 每个线程块的最大线程数 |
| `GPU_THREAD_MAX_REGISTERS` | 255 | 每个线程的最大寄存器数 |
| `GPU_KERNEL_BLOCK_NUM_THREADS` | 256 | 每个线程块的实际线程数（可调） |
| `GPU_KERNEL_MAX_REGISTERS` | 48/64 | 每个线程的寄存器限制（CUDA 9+且 arch>=600 时为 64） |

#### 架构 7.x/8.x/12.x（Volta/Ampere/Blackwell，`__CUDA_ARCH__ <= 1299`）
| 参数 | 值 | 说明 |
|------|----|------|
| `GPU_KERNEL_BLOCK_NUM_THREADS` | 384 | 较新架构使用更大的线程块 |
| `GPU_KERNEL_MAX_REGISTERS` | 168 | 较新架构允许更多寄存器 |

### 内核启动宏
- **`ccl_gpu_kernel(block_num_threads, thread_num_registers)`**: 定义带 `__launch_bounds__` 的全局内核函数，自动计算最小块数 = `总寄存器 / (线程数 * 每线程寄存器)`
- **`ccl_gpu_kernel_threads(block_num_threads)`**: 仅指定线程数的简化版本
- **`ccl_gpu_kernel_signature(name, ...)`**: 生成 `kernel_gpu_<name>` 格式的内核函数签名
- **`ccl_gpu_kernel_postfix`**: 内核函数后缀（CUDA 上为空）
- **`ccl_gpu_kernel_call(x)`**: 直接调用（无封装）
- **`ccl_gpu_kernel_within_bounds(i, n)`**: 边界检查 `i < n`

### Lambda 函数对象
- **`ccl_gpu_kernel_lambda(func, ...)`**: 定义 `KernelLambda` 结构体，捕获额外状态变量，重载 `operator()` 执行 func 表达式

### 编译时健全性检查
三项静态检查确保配置参数一致性：
1. 线程块线程数不超过硬件最大值
2. 计算出的块数不超过多处理器最大块数
3. 每线程寄存器不超过硬件最大值

## 依赖关系

- **内部头文件**: 无（自包含，依赖编译器预定义的 `__CUDA_ARCH__` 和 `__CUDACC_VER_MAJOR__`）
- **被引用**: `src/kernel/device/cuda/kernel.cu`

## 实现细节 / 关键算法

### GPU 占用率优化
`__launch_bounds__` 告知 CUDA 编译器每个线程块的线程数和每个多处理器的最小块数。编译器据此限制寄存器分配：若每线程使用寄存器过多，块数减少导致占用率下降。通过 `GPU_KERNEL_MAX_REGISTERS` 控制上限，平衡寄存器压力与占用率。

### CUDA 9+ Pascal 特殊处理
CUDA 9.0 编译器在 Pascal 架构（6.x）上表现出使用较少寄存器反而变慢的情况，因此将寄存器限制从 48 提升到 64，给予编译器更多寄存器分配空间。

### 架构分层
清晰的架构分层确保：
- 旧架构（Maxwell/Pascal）使用保守的 256 线程块和较少寄存器
- 新架构（Volta 及以后）使用更大的 384 线程块和更多寄存器
- 未知架构触发编译错误，防止在未验证的 GPU 上运行

## 关联文件

- `src/kernel/device/hip/config.h` — HIP 设备的对应配置（参数有差异）
- `src/kernel/device/cuda/compat.h` — CUDA 兼容层
- `src/kernel/device/gpu/kernel.h` — GPU 通用内核实现，使用这里定义的启动宏
