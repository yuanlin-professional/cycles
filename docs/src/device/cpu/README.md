# device/cpu - CPU 设备实现

## 概述

`cpu/` 子目录实现了 Cycles 的 CPU 渲染后端。`CPUDevice` 直接继承 `Device` 基类（非 `GPUDevice`），将路径追踪内核编译为本机代码执行。CPU 后端是唯一始终可用的后端，不依赖任何外部 GPU 驱动或运行时。

CPU 后端的核心特点：
- **SIMD 优化**：通过 `CPUKernelFunction` 模板在运行时自动选择最优指令集变体（默认或 AVX2）。
- **Embree 集成**：可选使用 Intel Embree 库进行硬件加速的光线-场景求交。
- **OSL 支持**：支持 Open Shading Language（OSL）着色器。
- **路径引导**：支持 Intel Open Path Guiding Library (OpenPGL) 进行路径引导。
- **Mega Kernel 模式**：CPU 后端使用 mega kernel（集成式内核），而非 GPU 的波前内核。

## 目录结构

| 文件名 | 类型 | 说明 |
|--------|------|------|
| `device.h` | 头文件 | `device_cpu_create()`、`device_cpu_info()`、`device_cpu_capabilities()` 工厂函数声明 |
| `device.cpp` | 源文件 | 工厂函数实现，CPU 设备信息与能力查询 |
| `device_impl.h` | 头文件 | `CPUDevice` 类定义 |
| `device_impl.cpp` | 源文件 | `CPUDevice` 实现：内存管理、BVH 构建、纹理加载、OSL/Embree/Guiding 集成 |
| `kernel.h` | 头文件 | `CPUKernels` 类定义——所有 CPU 端内核函数指针集合 |
| `kernel.cpp` | 源文件 | `CPUKernels` 构造，绑定各指令集变体的内核函数 |
| `kernel_function.h` | 头文件 | `CPUKernelFunction<T>` 模板——按微架构分发内核调用 |

## 核心类与数据结构

### CPUDevice

定义位置：`device_impl.h`

继承自 `Device`。关键成员：

| 成员 | 说明 |
|------|------|
| `kernel_globals` | `KernelGlobalsCPU` 实例，包含全局常量与纹理指针 |
| `texture_info` | 纹理描述信息向量 |
| `osl_globals` | OSL 全局状态（条件编译 `WITH_OSL`） |
| `embree_traversable` | Embree 场景句柄（条件编译 `WITH_EMBREE`） |
| `embree_device` | Embree 设备句柄 |
| `guiding_device` | OpenPGL 路径引导设备（条件编译 `WITH_PATH_GUIDING`） |

关键方法：

| 方法 | 说明 |
|------|------|
| `mem_alloc()` / `mem_free()` 等 | CPU 内存管理（直接使用主机指针，设备指针等于主机指针） |
| `const_copy_to()` | 将常量数据写入 `KernelGlobalsCPU` 对应字段 |
| `build_bvh()` | 构建 BVH 加速结构（BVH2 或 Embree） |
| `load_texture_info()` | 加载纹理信息到内核全局变量 |
| `get_cpu_kernel_thread_globals()` | 为每个渲染线程创建独立的内核全局变量副本 |
| `get_cpu_osl_memory()` | 返回 OSL 全局内存 |
| `get_guiding_device()` | 返回 OpenPGL 路径引导设备 |

### CPUKernels

定义位置：`kernel.h`

存储所有 CPU 端内核函数指针，按功能分组：
- **积分器内核**：`integrator_init_from_camera`、`integrator_init_from_bake`、`integrator_megakernel`
- **着色器求值**：`shader_eval_displace`、`shader_eval_background`、`shader_eval_curve_shadow_transparency`、`shader_eval_volume_density`
- **自适应采样**：`adaptive_sampling_convergence_check`、`adaptive_sampling_filter_x`、`adaptive_sampling_filter_y`
- **体积引导**：`volume_guiding_filter_x`、`volume_guiding_filter_y`
- **Cryptomatte**：`cryptomatte_postprocess`
- **胶片转换**：各 pass 类型的 `film_convert_xxx` 函数

### CPUKernelFunction\<T\>

定义位置：`kernel_function.h`

模板包装器，在构造时根据 CPU 能力（`system_cpu_support_avx2()`）自动选择最优的内核函数变体。当前支持的变体：
- `default` — 基线指令集
- `AVX2` — AVX2 优化版本（条件编译 `WITH_CYCLES_OPTIMIZED_KERNEL_AVX2`）

## 硬件要求

- **平台**：所有支持 Blender 的平台（Windows、Linux、macOS）
- **指令集**：基线为 SSE2（x86_64 默认），可选 AVX2 加速
- **Embree**：需要 Embree 4.x（条件编译 `WITH_EMBREE`），支持 `RTCTraversable`（Embree 4.4+）或 `RTCScene`
- **OSL**：需要 OpenShadingLanguage（条件编译 `WITH_OSL`）
- **路径引导**：需要 OpenPGL（条件编译 `WITH_PATH_GUIDING`）

## API 封装

CPU 后端不封装外部 GPU API，直接使用：
- 标准 C++ 内存管理
- Embree C API（`rtcore.h`）
- OpenShadingLanguage C++ API（`oslexec.h`）
- OpenPGL C++ API（`openpgl::cpp::Device`）

## 依赖关系

### 上游依赖（本模块依赖）

| 模块 | 用途 |
|------|------|
| `device/` | `Device` 基类、`device_memory` 体系 |
| `kernel/device/cpu/` | CPU 端内核实现（各指令集变体） |
| `kernel/` | 内核全局变量 (`KernelGlobalsCPU`)、OSL 全局变量 |
| `bvh/` | BVH 加速结构构建 |
| `util/` | 系统检测、线程、调试工具 |

### 下游依赖（依赖本模块）

| 模块 | 用途 |
|------|------|
| `device/device.cpp` | 工厂方法调用 `device_cpu_create()` |
| `session/` | 获取 CPU 内核函数 (`Device::get_cpu_kernels()`) |
| `integrator/` | 通过 `CPUKernels` 直接调用 mega kernel |

## 参见

- `src/device/device.h` — `Device` 基类
- `src/kernel/device/cpu/` — CPU 内核实现代码
- `src/device/multi/` — 多设备聚合（CPU+GPU 混合渲染）
