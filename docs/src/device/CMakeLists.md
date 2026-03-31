# CMakeLists.txt - 设备模块构建配置

## 概述

本文件是 Cycles 渲染器设备模块（`cycles_device`）的 CMake 构建配置。它定义了设备抽象层及全部后端（CPU、CUDA、HIP、HIPRT、Metal、oneAPI、OptiX、Dummy、Multi）的源文件组织、条件编译选项和外部库链接规则。通过编译选项宏控制各 GPU 后端的启用，是设备层的编译入口。

## 源文件组织

### SRC_BASE（基础抽象层）
位于 `src/device/` 根目录，包含设备抽象的核心文件：
- `device.cpp` — 设备基类与工厂实现
- `denoise.cpp` — 降噪器参数定义
- `graphics_interop.cpp` — 图形互操作接口
- `kernel.cpp` — 内核枚举与分类工具
- `memory.cpp` — 设备内存管理
- `queue.cpp` — 命令队列抽象

### SRC_HEADERS（公共头文件）
- `device.h`、`denoise.h`、`graphics_interop.h`、`memory.h`、`kernel.h`、`queue.h`

### 后端源文件组
- **SRC_CPU**: `cpu/` — CPU 设备实现（device、kernel、kernel_function）
- **SRC_CUDA**: `cuda/` — CUDA 设备实现（device、graphics_interop、kernel、queue、util）
- **SRC_HIP**: `hip/` — HIP 设备实现（与 CUDA 结构对称）
- **SRC_HIPRT**: `hiprt/` — HIP RT 硬件光线追踪扩展（device_impl、queue）
- **SRC_ONEAPI**: `oneapi/` — Intel oneAPI 设备实现（device、graphics_interop、queue）
- **SRC_DUMMY**: `dummy/` — 占位/回退设备实现
- **SRC_MULTI**: `multi/` — 多设备组合实现
- **SRC_METAL**: `metal/` — Apple Metal 设备实现（bvh、device、graphics_interop、kernel、queue、util，使用 `.mm` Objective-C++ 文件）
- **SRC_OPTIX**: `optix/` — NVIDIA OptiX 设备实现（device、queue、util）

## 条件编译与链接

### CUDA / OptiX
- 条件: `WITH_CYCLES_DEVICE_OPTIX` 或 `WITH_CYCLES_DEVICE_CUDA`
- 静态链接或动态加载: 通过 `WITH_CUDA_DYNLOAD` 控制，动态加载时链接 `extern_cuew`，否则链接 `CUDA_CUDA_LIBRARY`
- 定义宏: `CYCLES_CUDA_NVCC_EXECUTABLE`、`CYCLES_RUNTIME_OPTIX_ROOT_DIR`

### HIP
- 条件: `WITH_CYCLES_DEVICE_HIP`
- 动态加载: `WITH_HIP_DYNLOAD` 时链接 `extern_hipew`

### Metal
- 条件: `WITH_CYCLES_DEVICE_METAL`
- 链接: `METAL_LIBRARY`
- 将 `SRC_METAL` 追加到源文件列表

### oneAPI
- 条件: `WITH_CYCLES_DEVICE_ONEAPI`
- 链接: `cycles_kernel_oneapi` 库（区分 AOT `_aot` 和 JIT `_jit` 后缀）和 `SYCL_LIBRARIES`
- 可选宏: `WITH_ONEAPI_SYCL_HOST_TASK` 启用主机任务执行模式
- **Vulkan 互操作检测**: 使用 `check_cxx_source_compiles` 测试 SYCL 是否支持 `unmap_external_linear_memory` 函数，支持时定义 `SYCL_LINEAR_MEMORY_INTEROP_AVAILABLE` 宏

### OpenImageDenoise
- 条件: `WITH_OPENIMAGEDENOISE`
- 链接: `OPENIMAGEDENOISE_LIBRARIES`

### Open Shading Language
- 条件: `WITH_CYCLES_OSL`
- 链接: `OSL_LIBRARIES`

## 核心库依赖

- `cycles_kernel` — 内核实现库
- `cycles_util` — 工具库

## 构建目标

- `cycles_add_library(cycles_device ...)` — 创建 `cycles_device` 静态库
- oneAPI 后端添加 `add_dependencies(cycles_device cycles_kernel_oneapi)` 确保外部项目的正确重建

## Source Group 组织

使用 `source_group` 按后端名称组织 IDE 中的文件分组：`cpu`、`cuda`、`dummy`、`hip`、`hiprt`、`multi`、`metal`、`optix`、`oneapi`、`common`（基础文件和头文件）。

## 关联文件

- `src/device/` 下所有 `.cpp`、`.h`、`.mm` 源文件
- `src/kernel/` — 被链接为 `cycles_kernel` 依赖
- `src/util/` — 被链接为 `cycles_util` 依赖
