# denoiser.h / denoiser.cpp - 降噪器基类与工厂创建

## 概述

本文件定义了 Cycles 渲染器降噪系统的抽象基类 `Denoiser`，以及用于创建具体降噪器实例的工厂方法。它负责根据设备能力和用户配置参数，自动选择最优的降噪后端（OptiX、OIDN GPU 或 OIDN CPU），并提供统一的降噪接口。该文件是整个降噪子系统的入口和调度中心。

## 类与结构体

### Denoiser

- **继承**: 无（抽象基类）
- **功能**: 定义降噪器的通用接口，负责内核加载、参数管理和降噪缓冲区处理。通过工厂方法 `create()` 根据设备类型和参数创建具体的降噪器子类实例。
- **关键成员**:
  - `denoiser_device_` — 执行降噪运算的设备指针
  - `denoise_kernels_are_loaded_` — 标记降噪内核是否已加载
  - `params_` — 当前降噪参数（`DenoiseParams`）
  - `is_cancelled_cb` — 可选的取消回调函数，用于用户中断
- **关键方法**:
  - `create()` — 静态工厂方法，根据设备与参数创建最优降噪器实例（OptiX > OIDN GPU > OIDN CPU）
  - `denoise_buffer()` — 纯虚函数，对整个渲染缓冲区执行降噪操作
  - `load_kernels()` — 加载降噪所需的设备内核
  - `set_params()` / `get_params()` — 设置/获取降噪参数（不允许更改降噪器类型）
  - `automatic_viewport_denoiser_type()` — 根据设备信息推荐视口降噪类型
  - `get_denoiser_device()` — 获取实际执行降噪的设备
  - `get_device_type_mask()` — 纯虚函数，返回子类支持的设备类型掩码

## 核心函数

- `is_single_device(device)` — 静态辅助函数，判断设备是否为单设备（非 MultiDevice）
- `find_best_device(device, type, interop_device)` — 在多设备中选择最佳降噪设备，优先选择非 CPU 设备和支持图形互操作的设备
- `use_optix_denoiser(device, params)` — 判断是否应使用 OptiX 降噪器
- `use_gpu_oidn_denoiser(device, params)` — 判断是否应使用 OIDN GPU 降噪器
- `get_effective_denoise_params(device, cpu_fallback, params, interop, out_device)` — 计算有效降噪参数，在不支持 GPU 降噪时自动回退到 CPU OIDN

## 依赖关系

- **内部头文件**:
  - `device/denoise.h` — 降噪参数定义
  - `device/device.h` — 设备接口
  - `util/unique_ptr.h` — 智能指针
  - `integrator/denoiser_oidn.h` — OIDN CPU 降噪器
  - `integrator/denoiser_oidn_gpu.h` — OIDN GPU 降噪器（条件编译 `WITH_OPENIMAGEDENOISE`）
  - `integrator/denoiser_optix.h` — OptiX 降噪器（条件编译 `WITH_OPTIX`）
  - `session/buffers.h` — 渲染缓冲区
  - `session/display_driver.h` — 图形互操作设备定义
- **被引用**:
  - `integrator/denoiser_gpu.h`
  - `integrator/denoiser_oidn.h`
  - `integrator/path_trace.h`
  - `integrator/render_scheduler.h`
  - `session/denoising.h`

## 实现细节 / 关键算法

### 设备选择策略

`create()` 工厂方法的降噪器选择优先级：

1. 如果设备支持且用户指定 OptiX，创建 `OptiXDenoiser`
2. 如果设备支持且用户指定 OIDN GPU，创建 `OIDNDenoiserGPU`
3. 否则回退到 `OIDNDenoiser`（CPU OIDN）

`find_best_device()` 在多设备场景中遍历所有子设备，按以下规则选择最佳降噪设备：
- 优先非 CPU 设备（GPU 性能更优）
- 优先支持图形互操作的设备（减少显示更新延迟）

### 内核加载

`load_kernels()` 实现了惰性加载：已加载的内核不会重复加载。仅请求 `KERNEL_FEATURE_DENOISING` 特性，避免不必要的编译开销。

### 参数回退

`get_effective_denoise_params()` 在设备不支持所请求的降噪类型时，自动将参数降级为 `DENOISER_OPENIMAGEDENOISE` 且 `use_gpu = false`，确保始终有可用的降噪方案。

## 关联文件

- `src/integrator/denoiser_gpu.h/.cpp` — GPU 降噪基类
- `src/integrator/denoiser_oidn.h/.cpp` — OIDN CPU 降噪实现
- `src/integrator/denoiser_oidn_gpu.h/.cpp` — OIDN GPU 降噪实现
- `src/integrator/denoiser_optix.h/.cpp` — OptiX 降噪实现
- `src/device/denoise.h` — `DenoiseParams` 定义
- `src/session/buffers.h` — `RenderBuffers`、`BufferParams` 定义
