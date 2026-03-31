# kernel/device/gpu - GPU 通用内核基础设施

## 概述

`kernel/device/gpu/` 提供所有 GPU 后端（CUDA、HIP、HIPRT、OptiX、Metal、oneAPI）共享的内核基础设施代码。该目录并非一个独立的编译入口，而是被各平台的内核入口文件 `#include` 引用。其核心文件 `kernel.h` 统一定义了波前路径追踪（wavefront path tracing）的所有 GPU 内核函数，包括积分器各阶段的调度（intersect、shade、init）、胶片转换、自适应采样、着色器求值等。

GPU 内核采用波前并行架构：每个内核函数对应路径追踪流水线中的一个阶段，由主机端按需提交至 GPU 执行。线程以 block/wavefront 为单位执行，通过共享内存和 warp 级原语实现并行归约操作。

## 目录结构

| 文件名 | 类型 | 说明 |
|--------|------|------|
| `kernel.h` | 头文件 | GPU 内核函数统一定义，包含所有积分器阶段（init_from_camera, intersect_closest, shade_surface 等）、胶片转换、自适应采样等内核的 `ccl_gpu_kernel_signature` 声明与实现 |
| `block_sizes.h` | 头文件 | 定义 GPU 并行操作的默认 block 大小（HIP 使用 1024，其他平台使用 512），以及排序相关常量 |
| `work_stealing.h` | 头文件 | 工作窃取工具函数 `get_work_pixel()`，将全局工作索引映射为 tile 坐标 (x, y) 和采样索引 (sample) |
| `image.h` | 头文件 | GPU 纹理采样实现，包含双三次插值 (`kernel_tex_image_interp_bicubic`) 和标准纹理查找，通过 `ccl_gpu_tex_object_read_2D` 抽象各平台纹理 API |
| `parallel_active_index.h` | 头文件 | 并行活跃索引数组构建：给定状态数组，高效构建活跃状态的索引数组。使用 warp 级 ballot 和前缀和实现，各平台有不同的特化路径（Metal、oneAPI、CUDA/HIP） |
| `parallel_prefix_sum.h` | 头文件 | 并行前缀和实现（当前为串行版本，仅在单线程中执行），用于着色器数量级别的小数组 |
| `parallel_sorted_index.h` | 头文件 | 并行排序索引数组构建：按键值对活跃状态排序，支持本地原子排序优化路径 (`__KERNEL_LOCAL_ATOMIC_SORT__`)，包含 bucket pass 和 write pass 两阶段 |

## 内核函数入口

`kernel.h` 中通过 `ccl_gpu_kernel_signature` 宏定义了所有 GPU 内核函数，命名模式为 `kernel_gpu_<name>`。主要入口包括：

**积分器初始化：**
- `integrator_init_from_camera` -- 从相机生成初始光线
- `integrator_init_from_bake` -- 烘焙模式初始化

**光线相交：**
- `integrator_intersect_closest` -- 最近交点查询
- `integrator_intersect_shadow` -- 阴影光线查询
- `integrator_intersect_subsurface` -- 次表面散射交点查询
- `integrator_intersect_volume_stack` -- 体积栈交点查询
- `integrator_intersect_dedicated_light` -- 专用灯光交点查询

**着色：**
- `integrator_shade_background` -- 背景着色
- `integrator_shade_light` -- 灯光着色
- `integrator_shade_shadow` -- 阴影着色
- `integrator_shade_surface` / `shade_surface_raytrace` / `shade_surface_mnee` -- 表面着色（含光线追踪和 MNEE 变体）
- `integrator_shade_volume` / `shade_volume_ray_marching` -- 体积着色
- `integrator_shade_dedicated_light` -- 专用灯光着色

**路径状态管理：**
- `integrator_queued_paths_array` / `queued_shadow_paths_array` -- 队列路径数组
- `integrator_active_paths_array` / `terminated_paths_array` -- 活跃/终止路径数组
- `integrator_sorted_paths_array` -- 按着色器排序的路径数组
- `integrator_compact_paths_array` / `compact_states` -- 路径压缩
- `integrator_sort_bucket_pass` / `sort_write_pass` -- 本地原子排序

**胶片与后处理：**
- `film_convert_*` -- 各种胶片格式转换（depth, mist, combined, float4 等）
- `adaptive_sampling_convergence_check` / `filter_x` / `filter_y` -- 自适应采样
- `shader_eval_*` -- 着色器求值（背景、置换、曲线透明度、体积密度）
- `cryptomatte_postprocess` -- Cryptomatte 后处理

## GPU 兼容性

本目录为平台无关的通用代码，通过条件编译宏适配各平台：
- `__KERNEL_METAL__` -- Apple Metal 特化路径
- `__KERNEL_ONEAPI__` -- Intel oneAPI/SYCL 特化路径
- `__HIP__` -- AMD HIP 特化路径（block 大小使用 1024）
- 默认路径适用于 CUDA/OptiX

block 大小配置：
- HIP: 1024 线程/block
- 其他平台（CUDA、OptiX、Metal、oneAPI）: 512 线程/block
- 排序 block 大小: 1024（所有平台统一）

## API 封装

本目录通过统一的抽象宏屏蔽底层 API 差异：
- `ccl_gpu_kernel_signature` -- 内核函数签名
- `ccl_gpu_global_id_x()` / `ccl_gpu_thread_idx_x` -- 线程索引
- `ccl_gpu_syncthreads()` / `ccl_gpu_local_syncthreads()` -- 线程同步
- `ccl_gpu_ballot()` -- warp 级投票
- `ccl_gpu_tex_object_read_2D` -- 纹理读取
- `ccl_gpu_shared` -- 共享内存限定符
- `ccl_gpu_kernel_lambda` -- 设备端 lambda 函数对象

这些宏在各平台的 `compat.h` 和 `config.h` 中定义为对应的原生 API 调用。

## 依赖关系

### 上游依赖（本模块依赖）
- `kernel/integrator/` -- 积分器各阶段的实际实现（init、intersect、shade）
- `kernel/film/` -- 胶片转换和自适应采样实现
- `kernel/bake/` -- 烘焙功能
- `kernel/tables.h` -- 常量表（在 Metal/oneAPI 上下文类作用域之前引入）
- `kernel/sample/lcg.h` -- LCG 随机数生成
- `util/atomic.h` -- 原子操作

### 下游依赖（依赖本模块）
- `kernel/device/cuda/kernel.cu` -- CUDA 内核入口
- `kernel/device/hip/kernel.cpp` -- HIP 内核入口
- `kernel/device/hiprt/kernel.cpp` -- HIPRT 内核入口
- `kernel/device/optix/kernel.cu` -- OptiX 内核入口（仅引用 `image.h`）
- `kernel/device/metal/kernel.metal` -- Metal 内核入口
- `kernel/device/oneapi/kernel.cpp` -- oneAPI 内核入口

## 参见

- `kernel/device/cuda/` -- CUDA 平台编译入口
- `kernel/device/hip/` -- HIP 平台编译入口
- `kernel/device/optix/` -- OptiX 平台编译入口（自定义光线追踪管线）
- `kernel/device/metal/` -- Metal 平台编译入口
- `kernel/device/oneapi/` -- oneAPI/SYCL 平台编译入口
- `kernel/integrator/` -- 波前路径追踪积分器实现
