# kernel/device/optix - OptiX 内核编译入口（NVIDIA 硬件光线追踪）

## 概述

`kernel/device/optix/` 是 Cycles 路径追踪引擎针对 NVIDIA OptiX 光线追踪框架的内核编译入口。OptiX 利用 NVIDIA RTX GPU 中的 RT Core 硬件加速单元来执行 BVH 遍历和光线-三角形求交，大幅提升渲染性能。

与标准 CUDA 后端不同，OptiX 采用**光线追踪管线（ray tracing pipeline）**架构：
- **Ray Generation 程序** (`__raygen__`) -- 发射光线，对应积分器各阶段
- **Closest Hit 程序** (`__closesthit__`) -- 记录最近交点信息
- **Any Hit 程序** (`__anyhit__`) -- 阴影测试、透明阴影、体积过滤、局部交点收集
- **Intersection 程序** (`__intersection__`) -- 自定义图元（曲线、点云）求交
- **Miss 程序** (`__miss__`) -- 光线未命中处理

OptiX 内核在 `__KERNEL_OPTIX__` 和 `__KERNEL_CUDA__` 双重标志下编译（OptiX 内核隐含为 CUDA 内核），并将所有函数强制内联以优化管线性能。

该目录还支持 OSL（Open Shading Language）着色器的 GPU 执行，通过独立的编译单元（`kernel_osl.cu`、`kernel_osl_camera.cu`）提供。

## 目录结构

| 文件名 | 类型 | 说明 |
|--------|------|------|
| `kernel.cu` | 源文件 | OptiX 主内核入口。引入兼容性和全局变量定义，包含积分器相交阶段代码。定义五个 `__raygen__` 程序：`integrator_intersect_closest`、`intersect_shadow`、`intersect_subsurface`、`intersect_volume_stack`、`intersect_dedicated_light` |
| `kernel_shader_raytrace.cu` | 源文件 | 着色器光线追踪内核。包含基础 `kernel.cu` 的全部内容，额外引入 `shade_surface.h`，定义 `__raygen__kernel_optix_integrator_shade_surface_raytrace` 和 `shade_surface_mnee` 程序。由于编译耗时较长，仅在场景需要时加载 |
| `kernel_osl.cu` | 源文件 | OSL 着色器支持内核。定义 `WITH_OSL`，包含 `kernel_shader_raytrace.cu` 及所有着色阶段（background、light、shadow、volume、dedicated_light），以及 `init_from_bake` 和胶片相关内核。提供完整的 OSL GPU 着色管线 |
| `kernel_osl_camera.cu` | 源文件 | OSL 相机内核。定义 `WITH_OSL`，包含 `init_from_camera` raygen 程序及工作窃取相关代码 |
| `compat.h` | 头文件 | OptiX 兼容性定义。定义 `__KERNEL_GPU__`、`__KERNEL_CUDA__`、`__KERNEL_OPTIX__`；所有 `ccl_device` 函数标记为 `static __device__ __forceinline__`（强制内联以提升管线性能）；包含 `<optix.h>`（通过 `OPTIX_DONT_INCLUDE_CUDA` 避免 CUDA 头文件冲突）；使用 `ccl_optional_struct_init = {}` 零初始化结构体帮助编译器优化 |
| `globals.h` | 头文件 | OptiX 全局数据结构。定义 `KernelParamsOptiX`（含路径索引数组、渲染缓冲区、tile 参数、场景数据、积分器状态和 OSL 颜色系统指针），存储于 `__constant__` 内存中。在非 RDC（relocatable device code）模式下声明为 `static` |
| `bvh.h` | 头文件 | OptiX BVH 场景求交实现。通过 `optixTrace()`/`optixTraverse()` 和 payload 寄存器（p0-p7）实现光线-场景求交。包含完整的命中/未命中程序、可见性测试、阴影记录、局部交点、体积检测，以及曲线 ribbon 和点云的自定义相交程序 |

## 内核函数入口

OptiX 内核使用光线追踪管线程序模型：

**Ray Generation 程序（积分器调度）：**
- `__raygen__kernel_optix_integrator_intersect_closest` -- 最近交点查询
- `__raygen__kernel_optix_integrator_intersect_shadow` -- 阴影光线查询
- `__raygen__kernel_optix_integrator_intersect_subsurface` -- 次表面散射查询
- `__raygen__kernel_optix_integrator_intersect_volume_stack` -- 体积栈查询
- `__raygen__kernel_optix_integrator_intersect_dedicated_light` -- 专用灯光查询
- `__raygen__kernel_optix_integrator_shade_surface_raytrace` -- 表面光线追踪着色
- `__raygen__kernel_optix_integrator_shade_surface_mnee` -- MNEE 着色
- `__raygen__kernel_optix_integrator_shade_background` -- 背景着色（OSL）
- `__raygen__kernel_optix_integrator_shade_light` -- 灯光着色（OSL）
- `__raygen__kernel_optix_integrator_init_from_camera` -- 相机初始化（OSL）

**Hit 程序（bvh.h）：**
- `__closesthit__kernel_optix_hit` -- 记录最近交点（三角形、曲线、点云）
- `__anyhit__kernel_optix_visibility_test` -- 可见性测试（含阴影早终止）
- `__anyhit__kernel_optix_shadow_all_hit` -- 透明阴影记录
- `__anyhit__kernel_optix_local_hit` -- 局部交点收集（SSS）
- `__anyhit__kernel_optix_volume_test` -- 体积对象过滤
- `__anyhit__kernel_optix_ignore` -- 忽略交点
- `__closesthit__kernel_optix_ignore` -- 忽略最近交点（空操作）
- `__miss__kernel_optix_miss` -- 未命中处理

**Intersection 程序：**
- `__intersection__curve_ribbon` -- 曲线 ribbon 自定义求交
- `__intersection__point` -- 点云自定义求交

**Shader Binding Table (SBT) 偏移：**
- 偏移 0: `PG_HITD` -- 默认命中组
- 偏移 1: `PG_HITS` -- 阴影命中组
- 偏移 2: `PG_HITL` -- 局部命中组
- 偏移 3: `PG_HITV` -- 体积命中组

## GPU 兼容性

**支持的硬件：**
- NVIDIA Volta 及更新架构（SM 7.0+）
- 支持 RT Core 的 GPU 可获得硬件加速（Turing RTX 20xx 及更新）
- 无 RT Core 的 GPU 使用软件 BVH 遍历（回退到 CUDA 内核执行）

**OptiX 管线特性：**
- 使用 8 个 payload 寄存器（p0-p7）传递交点数据
- `OPTIX_RAY_FLAG_ENFORCE_ANYHIT` 确保 any-hit 程序始终被调用
- `OPTIX_RAY_FLAG_TERMINATE_ON_FIRST_HIT` 用于不透明阴影的早终止
- 支持运动模糊（通过 `optixGetRayTime()`）
- 支持实例化（通过 `optixGetInstanceId()`）
- primitive 类型值限制 < 128（>= 128 保留给 OptiX 内部使用）

**编译模式：**
- 标准 PTX/cubin 编译
- 支持 RDC（relocatable device code）用于分离编译
- 支持 NVRTC 运行时编译

## API 封装

底层使用 **NVIDIA OptiX 7+ API**：

| 组件 | API |
|------|-----|
| 光线追踪 | `optixTrace()`, `optixTraverse()` |
| 启动索引 | `optixGetLaunchIndex()` |
| 命中信息 | `optixGetRayTmax()`, `optixGetTriangleBarycentrics()`, `optixGetPrimitiveIndex()` |
| 实例信息 | `optixGetInstanceId()`, `optixGetInstanceIdFromHandle()` |
| 命中判断 | `optixIsTriangleHit()`, `optixGetHitKind()`, `optixHitObjectIsHit()` |
| 命中控制 | `optixIgnoreIntersection()`, `optixTerminateRay()` |
| 自定义相交 | `optixReportIntersection()` |
| Payload | `optixSetPayload_N()`, `optixGetPayload_N()` (N=0..7) |
| 属性 | `optixGetAttribute_0()`, `optixGetAttribute_1()` |
| 光线参数 | `optixGetObjectRayOrigin()`, `optixGetObjectRayDirection()` |
| BVH 句柄 | `optixGetTransformListHandle()` |

同时继承 CUDA 的全部基础 API 封装。

## 依赖关系

### 上游依赖（本模块依赖）
- `kernel/device/gpu/image.h` -- GPU 纹理采样（使用 CUDA 纹理内置函数）
- `kernel/integrator/` -- 积分器各阶段实现
- `kernel/bvh/types.h`, `kernel/bvh/util.h` -- BVH 类型和工具
- `<optix.h>` -- OptiX API 头文件
- `<optix_function_table.h>` -- OptiX 函数表定义
- `kernel/types.h`, `kernel/tables.h` -- 内核类型和常量表

### 下游依赖（依赖本模块）
- `device/optix/` -- OptiX 设备主机端实现，构建管线、SBT 和加速结构后调度内核

## 参见

- `kernel/device/cuda/` -- 标准 CUDA 后端（无硬件光线追踪）
- `kernel/device/gpu/` -- GPU 通用内核基础设施
- `kernel/device/hiprt/` -- HIPRT 后端（AMD 硬件光线追踪，功能对等）
- `device/optix/` -- OptiX 设备主机端代码
