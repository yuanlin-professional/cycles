# atomic.h - 跨平台原子操作抽象层

## 概述

`atomic.h` 为 Cycles 渲染器提供统一的原子操作接口，覆盖 CPU 和所有 GPU 后端（CUDA、HIP、Metal、oneAPI）。在 CPU 端封装 Blender 的 `atomic_ops` 库，在 GPU 端为每个平台提供原生原子操作实现。这些原子操作主要用于内核中的并行计数器、光线追踪中的浮点累加等场景。

## 类与结构体

### GPU 端辅助函数（非类）

在 CUDA/HIP 平台上定义了 `atomic_compare_and_swap_float`，使用联合体 (union) 将浮点数转换为整数进行 CAS 操作。

在 Metal 平台上，为 `device` 和 `threadgroup` 两种地址空间分别提供模板化的原子操作函数。

## 核心函数/宏定义

### CPU 端宏（非 GPU 内核）

| 宏 | 说明 |
|----|------|
| `atomic_add_and_fetch_float(p, x)` | 浮点原子加并返回新值 |
| `atomic_compare_and_swap_float(p, old, new)` | 浮点原子比较交换 |
| `atomic_fetch_and_inc_uint32(p)` | 无符号 32 位原子自增 |
| `atomic_fetch_and_dec_uint32(p)` | 无符号 32 位原子自减 |
| `CCL_LOCAL_MEM_FENCE` | 本地内存屏障标志（CPU 端为 0） |
| `ccl_barrier(flags)` | 内存屏障（CPU 端为空操作） |

### CUDA / HIP 端

| 宏/函数 | 说明 |
|---------|------|
| `atomic_add_and_fetch_float` | 基于 `atomicAdd` 实现 |
| `atomic_fetch_and_add_uint32` | 基于 `atomicAdd` |
| `atomic_fetch_and_sub_uint32` | 基于 `atomicSub` |
| `atomic_fetch_and_or_uint32` | 基于 `atomicOr` |
| `atomic_compare_and_swap_float` | 使用 union 技巧 + `atomicCAS` |
| `ccl_barrier(flags)` | 映射到 `__syncthreads()` |

### Metal 端

为 `device` 和 `threadgroup` 地址空间提供模板函数：
- `atomic_fetch_and_add_uint32` / `atomic_fetch_and_sub_uint32`
- `atomic_fetch_and_inc_uint32` / `atomic_fetch_and_dec_uint32`
- `atomic_fetch_and_or_uint32`
- `atomic_add_and_fetch_float`（Metal 3.0+ 使用原生 `atomic_float`，旧版本使用 CAS 循环）
- `atomic_compare_and_swap_float`
- `atomic_store` / `atomic_fetch` / `atomic_store_local` / `atomic_load_local`

### oneAPI (SYCL) 端

使用 `sycl::atomic_ref` 实现所有原子操作，支持 `global_device_space` 和 `local_space` 两种地址空间：
- `atomic_fetch_and_add_uint32` / `atomic_fetch_and_sub_uint32`（unsigned int 和 int 重载）
- `atomic_fetch_and_add_uint32_shared` - 共享内存版本
- `atomic_add_and_fetch_float` / `atomic_compare_and_swap_float`
- `atomic_store_local` / `atomic_load_local`

## 依赖关系

- **内部头文件**: 无（CPU 端引用外部 `atomic_ops.h`）
- **外部依赖**: Blender `atomic_ops.h`（CPU 端）
- **被引用**: `util/stats.h`, `kernel/film/light_passes.h`, `kernel/device/gpu/parallel_active_index.h`, `kernel/device/gpu/parallel_prefix_sum.h`, `kernel/device/gpu/parallel_sorted_index.h`

## 实现细节

1. **CPU/GPU 分离**：通过 `__KERNEL_GPU__` 宏区分 CPU 和 GPU 路径。CPU 端直接封装 Blender 的 `atomic_ops` 库；GPU 端为每个后端提供原生实现。

2. **浮点 CAS 技巧**：在不支持原生浮点原子操作的平台上，使用 union 将 `float` 重新解释为 `unsigned int`，再利用整数 CAS 操作实现浮点原子操作。Metal 3.0+ 已支持原生 `atomic_float`。

3. **内存序**：GPU 端所有原子操作均使用 `memory_order_relaxed`，因为 GPU 内核中的同步通过屏障 (`ccl_barrier`) 实现，不需要原子操作本身提供内存排序保证。

4. **地址空间**：Metal 和 oneAPI 需要区分全局内存 (`device`/`global`) 和本地共享内存 (`threadgroup`/`local`) 的原子操作，因此提供了独立的模板重载。

## 关联文件

- `util/defines.h` - 提供 `ccl_device_inline` 等 GPU 兼容宏
- `kernel/device/gpu/parallel_active_index.h` - 使用原子操作实现并行索引
