# kernel/device/oneapi - oneAPI/SYCL 内核编译入口（Intel GPU）

## 概述

`kernel/device/oneapi/` 是 Cycles 路径追踪引擎针对 Intel GPU（通过 oneAPI/SYCL 平台）的内核编译入口。oneAPI 是 Intel 的跨架构编程框架，其核心编程模型基于 SYCL（一种基于 C++ 的单源异构编程标准）。

oneAPI 后端的独特之处在于：
1. **动态链接库模式** -- 内核编译为独立的动态库（`.dll` / `.so`），通过 C 接口（`extern "C"`）导出，运行时由主机端动态加载
2. **SYCL 并行模型** -- 使用 `sycl::nd_range` 和 `sycl::parallel_for` 替代 CUDA 的网格/block 模型
3. **上下文类封装** -- 类似 Metal，通过 `ONEAPIKernelContext` 类封装设备资源访问
4. **Embree GPU 支持** -- 可选的硬件光线追踪加速，通过 Intel Embree 的 GPU 后端（SYCL 版本）实现
5. **模板化参数传递** -- 通过 `kernel_templates.h` 中的宏生成模板，自动将 `void**` 参数数组转换为类型安全的内核调用

oneAPI 后端同时支持标准设备执行和主机任务执行（`WITH_ONEAPI_SYCL_HOST_TASK`，用于调试）。

## 目录结构

| 文件名 | 类型 | 说明 |
|--------|------|------|
| `kernel.cpp` | 源文件 | oneAPI 内核编译入口和调度器。包含 SYCL 头文件、兼容性定义、全局变量和 GPU 通用内核代码。实现 `oneapi_enqueue_kernel()` 函数（巨型 switch-case 将 `DeviceKernel` 枚举映射为对应的 SYCL 内核提交）、`oneapi_load_kernels()`（JIT 编译和内核预加载）、`oneapi_run_test_kernel()`（设备功能验证）等导出函数。支持 Embree GPU 的特化常量设置 |
| `kernel.h` | 头文件 | oneAPI 内核公共接口声明。定义 `CYCLES_KERNEL_ONEAPI_EXPORT` 导出宏（跨平台 DLL 导出/导入）、`KernelContext` 结构体（队列、全局内存指针、最大着色器数）、以及所有导出函数的 C 链接声明：`oneapi_enqueue_kernel`、`oneapi_load_kernels`、`oneapi_run_test_kernel`、`oneapi_zero_memory_on_device`、`oneapi_set_error_cb`、`oneapi_suggested_gpu_kernel_size` |
| `compat.h` | 头文件 | oneAPI/SYCL 兼容性定义。定义 `__KERNEL_GPU__`、`__KERNEL_ONEAPI__`、`__KERNEL_64_BIT__`；将 Cycles 抽象映射为 SYCL 原语；使用 `sycl::ext::oneapi::this_work_item::get_nd_item<1>()` 获取线程索引；`ccl_gpu_kernel_signature` 宏生成命名为 `oneapi_kernel_<name>` 的函数，内部通过 `sycl::parallel_for(nd_range)` 提交内核；定义 SYCL 绑定纹理类型（`sycl::ext::oneapi::experimental::sampled_image_handle`）；支持 `WITH_ONEAPI_SYCL_HOST_TASK` 回退模式 |
| `globals.h` | 头文件 | oneAPI 全局数据结构。定义 `KernelGlobalsGPU`（含所有数据数组指针、积分器状态指针、`KernelData` 指针）。与其他 GPU 后端不同，oneAPI 无法使用 `__constant__` 全局变量，因此将所有常量数据指针存储在 `KernelGlobalsGPU` 结构体中。支持主机任务模式下的 `nd_item` 模拟字段以及标准模式下的 `sycl::kernel_handler` |
| `kernel_templates.h` | 头文件 | 类型安全的内核调用模板生成器。通过宏展开生成 0-21 参数的 `oneapi_call()` 模板函数，自动将 `void**` 参数数组中的元素转换（`*(T*)(args[i])`）为内核函数期望的类型。这是 oneAPI 后端实现 Cycles 通用内核调度接口的关键桥梁 |
| `context_begin.h` | 头文件 | oneAPI 上下文类开始标记。定义 `ONEAPIKernelContext` 类，封装 `KernelGlobalsGPU` 访问；在 Embree GPU 模式下声明特化常量 `oneapi_embree_features`；提供纹理采样和 NanoVDB 支持 |
| `context_end.h` | 头文件 | oneAPI 上下文类结束标记。关闭 `ONEAPIKernelContext` 类作用域 |
| `context_intersect_begin.h` | 头文件 | Embree GPU 相交上下文开始标记。在 `ONEAPIKernelContext` 中添加 Embree GPU 相交函数 |
| `context_intersect_end.h` | 头文件 | Embree GPU 相交上下文结束标记 |

## 内核函数入口

oneAPI 内核通过两层间接调用：

**外层：C 导出接口**
```cpp
bool oneapi_enqueue_kernel(KernelContext *ctx, int kernel, size_t global_size,
                           size_t local_size, uint kernel_features,
                           bool use_hardware_raytracing, void **args);
```

**内层：SYCL 内核函数**
```cpp
void oneapi_kernel_<name>(KernelGlobalsGPU *kg, size_t global_size,
                          size_t local_size, sycl::handler &cgh, ...);
```

每个 `oneapi_kernel_<name>` 函数内部通过 `cgh.parallel_for(sycl::nd_range<1>(...))` 提交 SYCL 内核。

**支持的全部设备内核：**
- 积分器：`integrator_reset`, `init_from_camera`, `init_from_bake`, 所有 `intersect_*` 和 `shade_*` 变体
- 路径管理：`queued_paths_array`, `active_paths_array`, `terminated_paths_array`, `sorted_paths_array`, `compact_*`
- 排序：`sort_bucket_pass`, `sort_write_pass`（使用 `sycl::local_accessor<int>` 分配共享内存）
- 自适应采样：`convergence_check`, `filter_x`, `filter_y`
- 着色器求值：`displace`, `background`, `curve_shadow_transparency`, `volume_density`
- 胶片转换：所有 `film_convert_*` 变体
- 后处理：`cryptomatte_postprocess`, `filter_guiding_*`, `filter_color_*`
- 体积引导：`volume_guiding_filter_x/y`

**按需加载机制：**
`oneapi_kernel_is_required_for_features()` 根据场景特性（`kernel_features`）跳过不需要的内核编译。

## GPU 兼容性

**支持的硬件：**
- Intel Arc 独立显卡（Alchemist DG2 及更新）
- Intel 集成显卡（部分支持）
- 需要 Intel oneAPI 工具链和运行时

**Embree GPU 加速：**
- 可选的硬件光线追踪，通过 `WITH_EMBREE_GPU` 编译标志启用
- 使用 Embree 4 的 SYCL 后端，通过 `RTCFeatureFlags` 特化常量控制启用的几何特性
- Embree 4.1+ 支持 MNEE 和 Ray-trace 内核的硬件加速
- 不兼容硬件光线追踪的内核会使用软件回退路径

**建议的 block 大小：**
| 内核类别 | block 大小 |
|----------|-----------|
| 并行活跃索引数组 | 512 |
| 并行排序索引数组 | 512 |
| 前缀和 | 512 |
| 排序 bucket/write pass | 1024 |
| 其他内核 | 0（由运行时决定） |

**JIT 编译：**
- 首次执行时通过 `sycl::get_kernel_bundle<sycl::bundle_state::executable>()` 触发 JIT 编译
- Embree GPU 内核通过 `sycl::build()` 使用特化常量进行 JIT 编译
- 支持 AOT（Ahead-of-Time）编译的缓存二进制

## API 封装

底层使用 **SYCL 2020** 和 **Intel oneAPI 扩展**：

| Cycles 抽象 | SYCL/oneAPI 实现 |
|-------------|-----------------|
| `ccl_device` | `inline` |
| `ccl_global` | （空，SYCL USM 统一内存） |
| `ccl_gpu_shared` | （空，通过 `sycl::local_accessor` 分配） |
| `ccl_gpu_thread_idx_x` | `this_work_item::get_nd_item<1>().get_local_id(0)` |
| `ccl_gpu_block_dim_x` | `this_work_item::get_nd_item<1>().get_local_range(0)` |
| `ccl_gpu_global_id_x()` | `this_work_item::get_nd_item<1>().get_global_id(0)` |
| `ccl_gpu_syncthreads()` | `this_work_item::get_nd_item<1>().barrier()` |
| `ccl_gpu_local_syncthreads()` | `barrier(sycl::access::fence_space::local_space)` |
| `ccl_gpu_ballot(pred)` | `group_ballot(sub_group, pred).count()` |
| 纹理采样 | `sycl::ext::oneapi::experimental::sample_image<T>()` |
| 快速数学 | `sycl::native::cos/sin/exp/log/sqrt/...` |
| 内核提交 | `sycl::handler::parallel_for(nd_range<1>)` |
| 内存分配 | `sycl::malloc_device`, `sycl::aligned_alloc_host` |

**特化常量：**
- `ONEAPIKernelContext::oneapi_embree_features` -- 控制 Embree GPU 启用的几何特性

## 依赖关系

### 上游依赖（本模块依赖）
- `kernel/device/gpu/kernel.h` -- GPU 通用内核定义
- `kernel/device/gpu/image.h` -- GPU 纹理采样
- `kernel/integrator/` -- 积分器实现
- `kernel/device/cpu/bvh.h` -- Embree BVH 实现（与 CPU 后端共享）
- `device/kernel.cpp` -- 设备内核枚举和工具函数
- `<sycl/sycl.hpp>` -- SYCL 标准库
- Embree 4（可选）-- GPU 硬件光线追踪

### 下游依赖（依赖本模块）
- `device/oneapi/` -- oneAPI 设备主机端实现，动态加载编译后的内核库

## 参见

- `kernel/device/gpu/` -- GPU 通用内核基础设施
- `kernel/device/cpu/bvh.h` -- Embree BVH 实现（共享）
- `kernel/device/metal/` -- Metal 后端（类似的上下文类封装模式）
- `device/oneapi/` -- oneAPI 设备主机端代码
