# queue.h / queue.cpp - OptiX 设备命令队列

## 概述

本文件实现了 OptiX 设备的命令队列，负责将 Cycles 的设备内核调度请求分发到 OptiX 管线或回退到 CUDA 内核执行。`OptiXDeviceQueue` 是 OptiX 后端与 Cycles 积分器之间的核心调度层，决定每个内核是通过 OptiX 光线追踪管线（`optixLaunch`）执行还是通过普通 CUDA 内核启动。

## 类与结构体

### `OptiXDeviceQueue`
- **继承**: `CUDADeviceQueue`（公有继承）
- **功能**: 拦截内核入队请求，对需要 OptiX 管线的内核（光线求交及 OSL 着色）调用 `optixLaunch()`，其余内核回退到 `CUDADeviceQueue::enqueue()` 以普通 CUDA 内核方式执行。
- **关键方法**:
  - `OptiXDeviceQueue(OptiXDevice *device)` — 构造函数，传入关联的 `OptiXDevice` 实例
  - `init_execution()` — 初始化执行环境，调用基类的 `CUDADeviceQueue::init_execution()`
  - `enqueue(DeviceKernel kernel, int work_size, const DeviceKernelArguments &args)` — 核心调度方法，根据内核类型选择执行路径

## 核心函数

### `is_optix_specific_kernel()` (静态)
- **参数**: `DeviceKernel kernel`, `bool osl_shading`, `bool osl_camera`
- **返回值**: `bool`
- **功能**: 判断给定内核是否需要通过 OptiX 管线执行。满足以下任一条件返回 `true`：
  - 内核包含光线求交操作（`device_kernel_has_intersection(kernel)`）
  - 启用了 OSL 着色且内核涉及着色（`device_kernel_has_shading(kernel)`）
  - 启用了 OSL 相机且内核为 `DEVICE_KERNEL_INTEGRATOR_INIT_FROM_CAMERA`

### `OptiXDeviceQueue::enqueue()`
- **功能**: 内核调度的核心实现，执行流程如下：
  1. 调用 `is_optix_specific_kernel()` 判断是否需要 OptiX 管线；若不需要，回退到 `CUDADeviceQueue::enqueue()`
  2. 通过 CUDA 内存拷贝设置 `KernelParamsOptiX` 中的启动参数（`path_index_array`、`render_buffer`、`offset`、`num_tiles`、`max_tile_work_size`），不同内核设置不同参数子集
  3. 同步 CUDA 流以确保参数就绪
  4. 根据 `DeviceKernel` 类型选择对应的 OptiX 管线（`PIP_SHADE` 或 `PIP_INTERSECT`）和光线生成程序记录
  5. 构建完整的 `OptixShaderBindingTable`，设置 miss、hitgroup 和 callables 记录
  6. 若启用 OSL，追加 OSL 可调用程序数量到 `callablesRecordCount`
  7. 调用 `optixLaunch()` 启动光线追踪

### 内核到管线的映射关系

| 内核类别 | 管线 | 光线生成程序 |
|---------|------|-------------|
| `INTEGRATOR_INTERSECT_CLOSEST` | `PIP_INTERSECT` | `PG_RGEN_INTERSECT_CLOSEST` |
| `INTEGRATOR_INTERSECT_SHADOW` | `PIP_INTERSECT` | `PG_RGEN_INTERSECT_SHADOW` |
| `INTEGRATOR_INTERSECT_SUBSURFACE` | `PIP_INTERSECT` | `PG_RGEN_INTERSECT_SUBSURFACE` |
| `INTEGRATOR_INTERSECT_VOLUME_STACK` | `PIP_INTERSECT` | `PG_RGEN_INTERSECT_VOLUME_STACK` |
| `INTEGRATOR_INTERSECT_DEDICATED_LIGHT` | `PIP_INTERSECT` | `PG_RGEN_INTERSECT_DEDICATED_LIGHT` |
| `INTEGRATOR_SHADE_BACKGROUND` | `PIP_SHADE` | `PG_RGEN_SHADE_BACKGROUND` |
| `INTEGRATOR_SHADE_LIGHT` | `PIP_SHADE` | `PG_RGEN_SHADE_LIGHT` |
| `INTEGRATOR_SHADE_SURFACE` | `PIP_SHADE` | `PG_RGEN_SHADE_SURFACE` |
| `INTEGRATOR_SHADE_SURFACE_RAYTRACE` | `PIP_SHADE` | `PG_RGEN_SHADE_SURFACE_RAYTRACE` |
| `INTEGRATOR_SHADE_SURFACE_MNEE` | `PIP_SHADE` | `PG_RGEN_SHADE_SURFACE_MNEE` |
| `INTEGRATOR_SHADE_VOLUME` | `PIP_SHADE` | `PG_RGEN_SHADE_VOLUME` |
| `INTEGRATOR_SHADE_VOLUME_RAY_MARCHING` | `PIP_SHADE` | `PG_RGEN_SHADE_VOLUME_RAY_MARCHING` |
| `INTEGRATOR_SHADE_SHADOW` | `PIP_SHADE` | `PG_RGEN_SHADE_SHADOW` |
| `INTEGRATOR_SHADE_DEDICATED_LIGHT` | `PIP_SHADE` | `PG_RGEN_SHADE_DEDICATED_LIGHT` |
| `SHADER_EVAL_DISPLACE` | `PIP_SHADE` | `PG_RGEN_EVAL_DISPLACE` |
| `SHADER_EVAL_BACKGROUND` | `PIP_SHADE` | `PG_RGEN_EVAL_BACKGROUND` |
| `SHADER_EVAL_CURVE_SHADOW_TRANSPARENCY` | `PIP_SHADE` | `PG_RGEN_EVAL_CURVE_SHADOW_TRANSPARENCY` |
| `SHADER_EVAL_VOLUME_DENSITY` | `PIP_SHADE` | `PG_RGEN_EVAL_VOLUME_DENSITY` |
| `INTEGRATOR_INIT_FROM_CAMERA` | `PIP_SHADE` | `PG_RGEN_INIT_FROM_CAMERA` |

## 依赖关系

- **内部头文件**:
  - `device/optix/queue.h` — 自身头文件
  - `device/optix/device_impl.h` — 访问 `OptiXDevice` 成员（管线、SBT、启动参数等）
  - `device/cuda/queue.h` — 基类 `CUDADeviceQueue`
  - `kernel/device/optix/globals.h` — `KernelParamsOptiX` 结构定义
- **被引用**:
  - `src/device/optix/device_impl.cpp` — `gpu_queue_create()` 中实例化本类

## 实现细节 / 关键算法

1. **双路径调度**: 内核分为两条执行路径 -- OptiX 路径（光线求交 + OSL 着色）和 CUDA 路径（其余所有内核）。这种设计允许 OptiX 仅处理其擅长的硬件加速光线追踪，而通用计算任务仍由 CUDA 内核高效处理。

2. **启动参数按需设置**: 不同内核仅设置各自需要的启动参数子集，通过 `cuMemcpyHtoDAsync` 异步写入设备端 `KernelParamsOptiX` 结构的对应偏移处。这避免了每次调用时传输完整参数结构的开销。

3. **SBT 统一配置**: 所有 OptiX 启动共享相同的 miss/hitgroup/callable 记录基地址和步长，通过预先打包好的 `sbt_data` 实现。不同内核仅通过 `raygenRecord` 选择不同的光线生成程序入口。

4. **OSL 可调用扩展**: 当启用 OSL 时，`callablesRecordCount` 动态增加 OSL 程序组数量，使 OptiX 运行时能够识别并调用 OSL 着色器的 direct callable 程序。

5. **流同步**: 在 `optixLaunch()` 之前调用 `cuStreamSynchronize()` 确保所有异步参数拷贝完成，保证内核获取到正确的启动参数。

## 关联文件

- `src/device/optix/device_impl.h` / `device_impl.cpp` — `OptiXDevice` 类定义，提供管线和 SBT 数据
- `src/device/cuda/queue.h` / `queue.cpp` — CUDA 命令队列基类
- `src/kernel/device/optix/globals.h` — `KernelParamsOptiX` 启动参数布局
