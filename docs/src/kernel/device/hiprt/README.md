# kernel/device/hiprt - HIPRT 内核编译入口（AMD 硬件光线追踪）

## 概述

`kernel/device/hiprt/` 是 Cycles 路径追踪引擎针对 AMD GPU 硬件光线追踪加速器（HIP RT）的内核编译入口。HIPRT 是 AMD 提供的光线追踪库，可利用 RDNA 2/3 架构中的光线加速器（Ray Accelerator）硬件单元来加速 BVH 遍历和光线-场景求交。

与标准 HIP 后端不同，HIPRT 后端需要：
1. 自定义 BVH 遍历逻辑（基于 `hiprtScene` 和 `hiprtGeometry`）
2. 自定义相交函数（曲线、运动三角形、点云）
3. 自定义过滤函数（最近交点、阴影、局部、体积）
4. 额外的内核入口（带有 `hiprtGlobalStackBuffer` 参数的相交内核）
5. 共享内存中的遍历栈管理

HIPRT 后端复用 HIP 的 `compat.h` 和 `config.h`，但提供独立的 `globals.h` 以定义 HIPRT 特有的全局数据结构和函数表索引。

## 目录结构

| 文件名 | 类型 | 说明 |
|--------|------|------|
| `kernel.cpp` | 源文件 | HIPRT 内核编译入口。引入 HIP `compat.h`/`config.h`、HIPRT 设备实现 (`hiprt/impl/hiprt_device_impl.h`)、HIPRT `globals.h`，以及 GPU 通用 `image.h` 和 `kernel.h` |
| `globals.h` | 头文件 | HIPRT 全局数据结构和常量。定义 `KernelGlobalsGPU`（含 `hiprtGlobalStackBuffer` 和 `hiprtSharedStackBuffer`）、`KernelParamsHIPRT`（含函数表 `hiprtFuncTable`）；定义遍历栈大小常量（`HIPRT_THREAD_STACK_SIZE=64`、`HIPRT_SHARED_STACK_SIZE=24`、`HIPRT_THREAD_GROUP_SIZE=256`）；定义自定义相交和过滤函数表索引枚举 |
| `common.h` | 头文件 | HIPRT 通用结构体和宏。定义 `RayPayload`、`ShadowPayload`、`LocalPayload` 负载结构；定义 `SET_HIPRT_RAY`、`GET_TRAVERSAL_STACK`、`GET_TRAVERSAL_ANY_HIT`、`GET_TRAVERSAL_CLOSEST_HIT` 遍历宏；实现自定义相交函数（curve、motion_triangle、point）和过滤函数（closest、shadow、local、volume）以及全局分发函数 `intersectFunc` / `filterFunc` |
| `bvh.h` | 头文件 | HIPRT BVH 场景求交接口。实现 `scene_intersect`（最近交点）、`scene_intersect_shadow`（阴影）、`scene_intersect_local`（局部/SSS）、`scene_intersect_shadow_all`（透明阴影）、`scene_intersect_volume`（体积）等函数，调用 HIPRT 遍历 API |
| `hiprt_kernels.h` | 头文件 | HIPRT 特有内核入口。定义带有 `hiprtGlobalStackBuffer stack_buffer` 参数的内核：`integrator_intersect_closest`、`intersect_shadow`、`intersect_subsurface`、`intersect_volume_stack`、`intersect_dedicated_light`、`shade_surface_raytrace`、`shade_surface_mnee`。这些内核使用 `HIPRT_INIT_KERNEL_GLOBAL()` 宏初始化共享内存遍历栈 |

## 内核函数入口

HIPRT 后端包含两类内核：

**通用 GPU 内核**（来自 `kernel/device/gpu/kernel.h`）：
- 所有非相交内核直接复用，如 `shade_background`、`shade_light`、`film_convert_*` 等

**HIPRT 特有相交内核**（来自 `hiprt_kernels.h`，需要遍历栈参数）：
- `kernel_gpu_integrator_intersect_closest(..., stack_buffer)` -- 最近交点（使用 HIPRT 硬件遍历）
- `kernel_gpu_integrator_intersect_shadow(..., stack_buffer)` -- 阴影
- `kernel_gpu_integrator_intersect_subsurface(..., stack_buffer)` -- 次表面散射
- `kernel_gpu_integrator_intersect_volume_stack(..., stack_buffer)` -- 体积栈
- `kernel_gpu_integrator_intersect_dedicated_light(..., stack_buffer)` -- 专用灯光
- `kernel_gpu_integrator_shade_surface_raytrace(..., stack_buffer)` -- 表面光线追踪着色
- `kernel_gpu_integrator_shade_surface_mnee(..., stack_buffer)` -- MNEE 着色

所有 HIPRT 内核使用 `GPU_HIPRT_KERNEL_BLOCK_NUM_THREADS` (1024) 作为 block 大小。

**自定义相交函数（通过函数表分发）：**
- `curve_custom_intersect` -- 曲线自定义相交
- `motion_triangle_custom_intersect` -- 运动三角形自定义相交
- `point_custom_intersect` -- 点云自定义相交

**过滤函数（通过函数表分发）：**
- `closest_intersection_filter` -- 最近交点过滤（自相交排除、shadow linking）
- `shadow_intersection_filter` -- 阴影过滤（透明阴影记录）
- `shadow_intersection_filter_curves` -- 曲线阴影过滤（透明度衰减）
- `local_intersection_filter` -- 局部过滤（SSS 交点收集）
- `volume_intersection_filter` -- 体积过滤（体积对象检查）

## GPU 兼容性

**支持的硬件：**
- AMD RDNA 2 及更新架构（含光线加速器硬件）
- 需要 HIP RT 库支持

**遍历栈配置：**
| 参数 | 值 | 说明 |
|------|-----|------|
| `HIPRT_THREAD_STACK_SIZE` | 64 | 每线程全局栈大小 |
| `HIPRT_SHARED_STACK_SIZE` | 24 | 每线程 LDS 栈大小（经验值） |
| `HIPRT_THREAD_GROUP_SIZE` | 256 | 相交内核工作组大小（低于默认 1024 以控制 LDS 用量） |

内核以 256 线程/工作组运行（而非标准 HIP 的 1024），因为 HIPRT 遍历需要大量共享内存（每线程 `HIPRT_SHARED_STACK_SIZE * sizeof(int)` 字节）。

**函数表结构：**
HIPRT 使用 `hiprtFuncTable` 将自定义相交和过滤函数组织为四个独立表：
- `table_closest_intersect` -- 最近交点遍历
- `table_shadow_intersect` -- 阴影遍历
- `table_local_intersect` -- 局部（SSS）遍历
- `table_volume_intersect` -- 体积遍历

每个表按几何类型（三角形、曲线、运动三角形、点云）和光线类型索引。

## API 封装

底层使用 **HIP RT API**：

| 组件 | API |
|------|-----|
| 场景遍历 | `hiprtSceneTraversalClosestCustomStack`, `hiprtSceneTraversalAnyHitCustomStack` |
| 几何遍历 | `hiprtGeomTraversalAnyHitCustomStack`, `hiprtGeomCustomTraversalAnyHitCustomStack` |
| 栈管理 | `hiprtGlobalStack`, `hiprtGlobalStackBuffer`, `hiprtSharedStackBuffer` |
| 实例栈 | `hiprtEmptyInstanceStack` |
| 场景句柄 | `hiprtScene`, `hiprtGeometry` |
| 函数表 | `hiprtFuncTable`, `hiprtFuncTableHeader` |
| 设备函数 | `HIPRT_DEVICE` 标记的 `intersectFunc` / `filterFunc` |

同时复用 HIP 的所有基本 API 封装（线程管理、内存模型等）。

## 依赖关系

### 上游依赖（本模块依赖）
- `kernel/device/hip/compat.h` -- HIP 兼容性定义
- `kernel/device/hip/config.h` -- HIP 启动边界配置
- `kernel/device/gpu/kernel.h` -- GPU 通用内核定义
- `kernel/device/gpu/image.h` -- GPU 纹理采样
- `hiprt/impl/hiprt_device_impl.h` -- HIPRT 设备端实现
- `kernel/integrator/` -- 积分器实现
- `kernel/geom/` -- 曲线、点云相交函数

### 下游依赖（依赖本模块）
- `device/hip/` -- HIP/HIPRT 设备主机端实现，构建 HIPRT 场景和函数表后调度内核

## 参见

- `kernel/device/hip/` -- 标准 HIP 后端（软件 BVH 遍历）
- `kernel/device/gpu/` -- GPU 通用内核基础设施
- `kernel/device/optix/` -- OptiX 后端（NVIDIA 硬件光线追踪，功能对等）
- `device/hip/` -- HIP 设备主机端代码
