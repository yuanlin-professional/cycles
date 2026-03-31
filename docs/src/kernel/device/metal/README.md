# kernel/device/metal - Metal 内核编译入口（Apple GPU）

## 概述

`kernel/device/metal/` 是 Cycles 路径追踪引擎针对 Apple Metal GPU 平台的内核编译入口。Metal 是 Apple 的底层图形和计算 API，用于在 macOS 上的 Apple Silicon（M1/M2/M3 等）和 AMD 独立显卡上执行 GPU 计算。

Metal 后端的核心设计特点是**上下文类封装（MetalKernelContext）**：所有内核函数都在 `MetalKernelContext` 类的成员方法中执行，通过该类访问全局资源绑定（纹理、加速结构、函数表等）。这是 Metal 编程模型的要求——Metal 着色器通过参数缓冲区（而非全局变量）访问设备资源。

Metal 后端支持两种模式：
1. **计算内核模式** -- 用于非光线追踪的通用 GPU 内核（着色、胶片转换等）
2. **MetalRT 模式** -- 使用 Metal Ray Tracing API 进行硬件加速的光线-场景求交

编译产出为 `.metallib` 格式，使用 Metal Shading Language (MSL) 编译器。Metal 还引入了 **Function Constants（函数常量）** 机制来替代传统的预处理器条件编译，实现内核参数的运行时特化。

## 目录结构

| 文件名 | 类型 | 说明 |
|--------|------|------|
| `kernel.metal` | Metal 源文件 | Metal 内核编译入口。引入 `compat.h`、`globals.h`、`function_constants.h`，然后包含 GPU 通用 `kernel.h`。在 `__METALRT__` 宏下定义所有 MetalRT 相交处理程序（三角形、曲线、点云的 intersection/filter 函数），包括可见性测试、阴影记录、局部交点收集和体积过滤 |
| `compat.h` | 头文件 | Metal 兼容性定义（最大的平台 compat 文件）。定义 `__KERNEL_GPU__` 和 `__KERNEL_METAL__`；包含 Metal 标准库（`<metal_stdlib>`, `<metal_atomic>`, `<simd/simd.h>`）；将 Cycles 抽象映射为 MSL 限定符（`device`, `threadgroup`, `thread`, `constant`, `ray_data`）；定义 `ccl_gpu_kernel_signature` 宏生成结构体+计算内核+`run` 方法的三段式模式；支持 `__METAL_GLOBAL_BUILTINS__`（macOS 14+）和传统入口参数模式；定义 MetalRT 类型（加速结构、相交函数表、intersector）；提供 `TextureParamsMetal` 和 `MetalAncillaries` 资源结构 |
| `globals.h` | 头文件 | Metal 全局数据结构。定义 `KernelParamsMetal`（含数据数组指针、积分器状态和 `KernelData`）；数据访问宏通过 `launch_params_metal` 引用（而非 CUDA 的 `kernel_params`）；`KernelGlobals` 为空结构体指针 |
| `function_constants.h` | 头文件 | Metal Function Constants 定义。通过 `kernel/data_template.h` 为 `KernelData` 的每个成员生成对应的函数常量（`[[function_constant(N)]]`），包括 `kernel_data_kernel_features`。允许 Metal 编译器在 JIT 时根据场景特性进行死代码消除 |
| `context_begin.h` | 头文件 | Metal 上下文类开始标记。定义 `MetalKernelContext` 类（含 `launch_params_metal` 和 `metal_ancillaries` 资源引用），提供纹理采样特化模板（`ccl_gpu_tex_object_read_2D<float4>` 和 `<float>`），通过 sampler 和 bindless texture 实现纹理访问。在 GPU 通用代码之前打开类作用域 |
| `context_end.h` | 头文件 | Metal 上下文类结束标记。关闭 `MetalKernelContext` 类作用域，重新定义 `kernel_integrator_state` 宏指向上下文实例 |
| `bvh.h` | 头文件 | MetalRT BVH 场景求交实现。通过 Metal Ray Tracing API 的 `intersector` 对象和 `intersection_function_table` 执行硬件加速遍历，使用 `ray_data` 地址空间的自定义 payload 结构在 intersection 函数之间传递数据 |

## 内核函数入口

Metal 内核采用独特的三段式结构：

```metal
// 1. 参数结构体
struct kernel_gpu_<name> {
    /* 参数成员 */
    void run(thread MetalKernelContext& context, ...) ccl_global const;
};

// 2. 计算内核入口
kernel void cycles_metal_<name>(...) {
    MetalKernelContext context(...);
    params_struct->run(context, ...);
}

// 3. run 方法实现
void kernel_gpu_<name>::run(...) { /* 实际内核代码 */ }
```

所有 GPU 通用内核通过此模式自动映射为 Metal 计算内核。

**MetalRT 相交处理程序（kernel.metal）：**

三角形相交：
- `__intersection__tri` -- 默认三角形可见性测试
- `__intersection__tri_shadow` -- 三角形阴影可见性测试
- `__intersection__tri_shadow_all` -- 三角形透明阴影记录
- `__intersection__volume_tri` -- 三角形体积过滤
- `__intersection__local_tri` / `local_tri_mblur` -- 局部三角形交点（SSS）
- `__intersection__local_tri_single_hit` / `single_hit_mblur` -- 单命中局部交点

曲线相交：
- `__intersection__curve` -- 曲线可见性测试
- `__intersection__curve_shadow` -- 曲线阴影测试
- `__intersection__curve_shadow_all` -- 曲线透明阴影记录

点云相交：
- `__intersection__point` -- 点云可见性测试（bounding box intersection）
- `__intersection__point_shadow` -- 点云阴影测试
- `__intersection__point_shadow_all` -- 点云透明阴影记录

## GPU 兼容性

**支持的硬件：**
- Apple Silicon (M1, M2, M3, M4 系列) -- 最佳性能
- AMD 独立显卡 (macOS) -- 基本支持
- 需要 macOS 12.0 (Monterey) 及更新版本

**Metal 特性：**
- MetalRT（Metal Ray Tracing）用于硬件加速 BVH 遍历
- Function Constants 用于运行时内核特化
- `__METAL_GLOBAL_BUILTINS__` -- macOS 14+ 的全局内置常量（无需通过入口参数传递 `thread_position_in_grid` 等）
- `__METALRT_MOTION__` -- 运动模糊支持
- `__METALRT_EXTENDED_LIMITS__` -- 扩展 BVH 限制
- Simdgroup 操作（ballot、exclusive_scan）替代 CUDA warp 操作

**线程模型：**
- 使用 `threadgroup` 替代 CUDA `__shared__`
- 使用 `simdgroup` 替代 CUDA warp
- `simd_ballot` / `simd_vote` 用于 warp 级投票
- `threadgroup_barrier` 用于线程同步

**地址空间：**
- `device` -- 全局内存
- `threadgroup` -- 共享内存
- `thread` -- 私有内存
- `constant` -- 常量内存
- `ray_data` -- MetalRT 光线数据

## API 封装

底层使用 **Metal Shading Language (MSL)** 和 **Metal Ray Tracing API**：

| Cycles 抽象 | Metal 实现 |
|-------------|-----------|
| `ccl_device` | （默认，无特殊限定符） |
| `ccl_global` | `device` |
| `ccl_gpu_shared` | `threadgroup` |
| `ccl_private` | `thread` |
| `ccl_constant` | `constant` |
| `ccl_ray_data` | `ray_data`（MetalRT 模式）/ `thread`（非 MetalRT） |
| `ccl_gpu_syncthreads()` | `threadgroup_barrier(mem_flags::mem_threadgroup)` |
| `ccl_gpu_ballot(pred)` | `(uint64_t)((simd_vote::vote_t)simd_ballot(pred))` |
| 纹理采样 | `texture2d<float>.sample(sampler, coord)` |
| 光线追踪 | `metalrt_intersector_type`、`metalrt_as_type` |

**资源绑定：**
- `KernelParamsMetal` -- 通过 `constant` 缓冲区传递（`[[buffer(1)]]`）
- `MetalAncillaries` -- 纹理数组、加速结构、交点函数表
- 采样器数组 -- 编译时定义的 8 种采样器组合

## 依赖关系

### 上游依赖（本模块依赖）
- `kernel/device/gpu/kernel.h` -- GPU 通用内核定义
- `kernel/device/gpu/image.h` -- GPU 纹理采样（通过上下文类内联）
- `kernel/device/gpu/work_stealing.h` -- 工作窃取
- `kernel/integrator/` -- 积分器实现
- `kernel/tables.h` -- 常量表（在上下文类作用域前引入）
- `kernel/data_template.h` -- 用于生成函数常量
- Metal 标准库（`<metal_stdlib>`, `<metal_atomic>` 等）

### 下游依赖（依赖本模块）
- `device/metal/` -- Metal 设备主机端实现，编译 .metallib 并调度内核

## 参见

- `kernel/device/gpu/` -- GPU 通用内核基础设施
- `kernel/device/optix/` -- OptiX 后端（NVIDIA 硬件光线追踪，功能对等）
- `device/metal/` -- Metal 设备主机端代码
