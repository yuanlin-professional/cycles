# kernel/device/cpu - CPU 内核编译入口

## 概述

`kernel/device/cpu/` 是 Cycles 路径追踪引擎的 CPU 设备内核编译入口。CPU 后端采用**巨内核（megakernel）**架构，与 GPU 后端的波前（wavefront）架构不同：CPU 后端将整条光线路径的追踪在单个函数中完成，而非拆分为多个阶段内核。

CPU 后端支持两种指令集优化变体：
1. **基础版本 (`cpu`)** -- 编译时启用 SSE4.2 指令集（x86-64 最低要求）
2. **AVX2 优化版本 (`cpu_avx2`)** -- 在支持 AVX2 的处理器上使用更高效的 SIMD 指令

运行时会根据 CPU 实际支持的指令集，动态选择最优的内核变体。每个线程拥有独立的 `ThreadKernelGlobalsCPU` 实例（从共享的 `KernelGlobalsCPU` 拷贝而来），以避免指针间接寻址的开销。

## 目录结构

| 文件名 | 类型 | 说明 |
|--------|------|------|
| `kernel.cpp` | 源文件 | CPU 基础版内核入口点。定义 SSE4.2 编译标志，设置 `KERNEL_ARCH=cpu`，包含 `kernel_arch_impl.h` 展开所有内核函数。同时提供 `kernel_const_copy()` 和 `kernel_global_memory_copy()` 用于常量和全局内存传输 |
| `kernel_avx2.cpp` | 源文件 | AVX2 优化内核入口点。设置 `KERNEL_ARCH=cpu_avx2`，启用 SSE/AVX/AVX2 全部指令集标志。若编译时未启用 `WITH_CYCLES_OPTIMIZED_KERNEL_AVX2`，则定义 `KERNEL_STUB` 生成桩函数 |
| `kernel.h` | 头文件 | CPU 内核公共接口声明。通过 `KERNEL_NAME_JOIN` 宏模板生成带架构后缀的函数名（如 `kernel_cpu_integrator_init_from_camera`、`kernel_cpu_avx2_integrator_init_from_camera`），声明积分器、胶片转换、着色器求值、自适应采样、Cryptomatte、体积引导等全部内核函数 |
| `kernel_arch.h` | 头文件 | 模板化内核声明，被 `kernel.h` 为每个架构变体 include 一次。使用 `KERNEL_FUNCTION_FULL_NAME` 宏展开为架构特定的函数签名 |
| `kernel_arch_impl.h` | 头文件 | 模板化内核实现。include 所有积分器和胶片处理代码，通过 `DEFINE_INTEGRATOR_INIT_KERNEL`、`DEFINE_INTEGRATOR_SHADE_KERNEL`、`KERNEL_FILM_CONVERT_FUNCTION` 等宏定义实际函数体。若定义了 `KERNEL_STUB`，则生成仅包含断言的桩函数 |
| `globals.h` | 头文件 | CPU 全局数据结构定义。`KernelGlobalsCPU` 包含场景数据数组和 `KernelData`；`ThreadKernelGlobalsCPU` 继承自前者并添加线程级 OSL 状态和路径引导（path guiding）支持。定义 `kernel_data_fetch` 等数据访问宏 |
| `globals.cpp` | 源文件 | `ThreadKernelGlobalsCPU` 构造函数实现，初始化 OSL 线程数据和 OpenPGL 路径段存储 |
| `compat.h` | 头文件 | CPU 兼容性定义。抑制 GCC 中 release 模式下的 `-Wmaybe-uninitialized` 警告；定义 `kernel_assert` 为标准 `assert()`（CPU 是唯一支持内核断言的设备） |
| `image.h` | 头文件 | CPU 纹理采样实现，支持线性和三次样条插值，使用命名空间 (`namespace {}`) 防止不同指令集变体间的符号冲突 |
| `bvh.h` | 头文件 | CPU BVH 光线相交实现，基于 Embree 4 (`embree4/rtcore_*.h`)。同时被 oneAPI 后端共享 |

## 内核函数入口

CPU 内核函数通过架构后缀的命名方式区分不同指令集变体：

**积分器（巨内核模式）：**
- `kernel_cpu[_avx2]_integrator_init_from_camera` -- 相机光线初始化
- `kernel_cpu[_avx2]_integrator_init_from_bake` -- 烘焙初始化
- `kernel_cpu[_avx2]_integrator_megakernel` -- 巨内核（完整路径追踪循环）

**胶片转换：**
- `kernel_cpu[_avx2]_film_convert_<type>` -- 各种输出格式转换（depth, mist, combined, float4 等）
- `kernel_cpu[_avx2]_film_convert_half_rgba_<type>` -- 半精度 RGBA 版本

**着色器求值：**
- `kernel_cpu[_avx2]_shader_eval_background` -- 背景求值
- `kernel_cpu[_avx2]_shader_eval_displace` -- 置换求值
- `kernel_cpu[_avx2]_shader_eval_curve_shadow_transparency` -- 曲线阴影透明度
- `kernel_cpu[_avx2]_shader_eval_volume_density` -- 体积密度求值

**自适应采样：**
- `kernel_cpu[_avx2]_adaptive_sampling_convergence_check` -- 收敛检查
- `kernel_cpu[_avx2]_adaptive_sampling_filter_x/y` -- 自适应采样滤波

**后处理：**
- `kernel_cpu[_avx2]_cryptomatte_postprocess` -- Cryptomatte 后处理
- `kernel_cpu[_avx2]_volume_guiding_filter_x/y` -- 体积引导滤波

## GPU 兼容性

本目录为 CPU 专用后端，不涉及 GPU 兼容性。

**CPU 指令集要求：**
- **最低要求：** x86-64 + SSE4.2
- **可选优化：** AVX2（运行时检测，需编译时启用 `WITH_CYCLES_OPTIMIZED_KERNEL_AVX2`）
- **原生编译模式：** 定义 `WITH_KERNEL_NATIVE` 时，从编译器内置宏自动检测可用指令集

**编译器支持：**
- GCC, Clang, MSVC (x86-64)
- 在 32 位 GCC 上禁用 SSE 优化（参见 bug #36316）

## API 封装

CPU 后端直接使用 C++ 标准调用约定，不需要 GPU 设备 API 封装。关键抽象层：

- **数据访问宏：** `kernel_data_fetch(name, index)` 展开为 `kg->name.fetch(index)`，带有边界检查
- **内核全局变量：** `KernelGlobals` 类型定义为 `const ThreadKernelGlobalsCPU *`，以指针形式传递
- **BVH 实现：** 通过 Embree 4 API 执行光线相交（与 oneAPI 后端共享 `bvh.h`）
- **纹理采样：** 通过 `kernel_array<T>` 模板直接访问内存，无需 GPU 纹理对象

## 依赖关系

### 上游依赖（本模块依赖）
- `kernel/integrator/` -- 积分器实现（`megakernel.h`, `init_from_camera.h`, `init_from_bake.h`）
- `kernel/film/` -- 胶片转换和自适应采样
- `kernel/bake/` -- 烘焙功能
- `kernel/types.h` -- 内核数据类型定义
- `kernel/data_arrays.h` -- 数据数组声明（通过 X-macro 模式展开）
- `kernel/osl/globals.h` -- OSL 全局数据（可选）
- Embree 4 -- BVH 光线相交
- OpenPGL -- 路径引导（可选，通过 `util/guiding.h`）

### 下游依赖（依赖本模块）
- `device/cpu/` -- CPU 设备主机端实现，调用本目录导出的内核函数
- `kernel/device/oneapi/` -- oneAPI 后端共享 `bvh.h`

## 参见

- `kernel/device/gpu/` -- GPU 通用内核基础设施（波前架构）
- `kernel/integrator/megakernel.h` -- CPU 巨内核积分器实现
- `kernel/device/oneapi/` -- oneAPI 后端（共享 Embree BVH 实现）
- `device/cpu/` -- CPU 设备主机端代码
