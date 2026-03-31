# denoiser_optix.h / denoiser_optix.cpp - OptiX 降噪器

## 概述

本文件实现了基于 NVIDIA OptiX 降噪 API 的 GPU 降噪器 `OptiXDenoiser`，继承自 `DenoiserGPU`。它利用 OptiX 内置的 AI 降噪模型（支持 AOV 和时序模型），在 NVIDIA GPU 上高效执行降噪运算。支持分块处理以应对大分辨率图像的显存限制。编译需启用 `WITH_OPTIX` 宏。

## 类与结构体

### OptiXDenoiser

- **继承**: `DenoiserGPU` -> `Denoiser`
- **功能**: 通过 OptiX Denoiser API 在 NVIDIA GPU 上执行降噪，支持 AOV（任意输出变量）和时序降噪模型。
- **关键成员**:
  - `optix_denoiser_` — OptiX 降噪器句柄（`OptixDenoiser`）
  - `is_configured_` / `configured_size_` — 配置状态和当前配置的瓦片尺寸
  - `state_` — 降噪器状态和临时缓冲区的设备内存（布局：[状态][临时空间]）
  - `sizes_` — OptiX 降噪器内存需求（`OptixDenoiserSizes`）
  - `use_pass_albedo_` / `use_pass_normal_` / `use_pass_motion_` — 引导通道启用标志
- **关键方法**:
  - `is_device_supported()` — 静态方法，检测设备是否为 OptiX 类型且支持 OptiX 降噪
  - `denoise_buffer(DenoiseTask)` — 在 CUDA 上下文作用域内调用基类降噪流程
  - `denoise_create_if_needed()` — 创建 OptiX 降噪器：选择 AOV 或 TEMPORAL 模型，配置 albedo/normal 引导选项
  - `denoise_configure_if_needed()` — 计算内存需求，分配状态缓冲区，调用 `optixDenoiserSetup()`
  - `denoise_run()` — 设置图层数据并通过分块调用执行降噪
  - `get_device_type_mask()` — 返回 `DEVICE_MASK_OPTIX`

## 核心函数

- `optixUtilDenoiserSplitImage()` — 将大图分割为重叠瓦片（OPTIX_ABI_VERSION < 60 的本地实现，修复了整数溢出问题）
- `optixUtilDenoiserInvokeTiled()` — 分块执行降噪（OPTIX_ABI_VERSION < 60 的本地实现），遍历瓦片并逐个调用 `optixDenoiserInvoke()`

## 依赖关系

- **内部头文件**:
  - `integrator/denoiser_gpu.h` — GPU 降噪基类
  - `device/optix/util.h` — OptiX 工具宏
  - `device/optix/device_impl.h` — OptiX 设备实现（获取 `OptixDeviceContext`）
  - `device/optix/queue.h` — OptiX 设备队列（获取 CUDA stream）
  - `integrator/pass_accessor_gpu.h` — GPU 渲染通道访问器
  - `<optix_denoiser_tiling.h>` — OptiX SDK 分块降噪工具
- **被引用**:
  - `integrator/denoiser.cpp` — 工厂方法中条件创建（`WITH_OPTIX`）

## 实现细节 / 关键算法

### 降噪模型选择

- **AOV 模型**（`OPTIX_DENOISER_MODEL_KIND_AOV`）: 默认模型，单帧降噪
- **时序模型**（`OPTIX_DENOISER_MODEL_KIND_TEMPORAL`）: 当启用运动引导通道（`use_pass_motion`）时使用，利用前后帧信息进行时序稳定的降噪

### 分块降噪

`denoise_configure_if_needed()` 将最大瓦片尺寸限制为 4096x4096。当图像超过此尺寸时：
1. 计算重叠窗口大小（`overlapWindowSizeInPixels`）
2. 将输入/输出图像分割为重叠瓦片
3. 将引导通道（albedo、normal、flow）同步分割
4. 逐瓦片调用 `optixDenoiserInvoke()` 执行降噪

### 整数溢出修复

当 OPTIX_ABI_VERSION < 60 时，文件包含了自定义的 `optixUtilDenoiserSplitImage()` 和 `optixUtilDenoiserInvokeTiled()` 实现，主要修复了原始 SDK 代码中的整数溢出问题（在行跨步与偏移量计算中使用 `size_t` 替代 `int`）。

### CUDA 上下文管理

`denoise_buffer(DenoiseTask)` 通过 `CUDAContextScope` RAII 对象确保在正确的 CUDA 上下文中执行降噪操作。

### 内存管理

降噪器状态和临时缓冲区存储在单一的 `device_only_memory` 中，布局为 `[denoiser state][scratch buffer]`。配置变更时重新分配。`optixDenoiserSetup()` 在默认 CUDA stream (0) 上执行，以避免 r495 驱动的已知渲染伪影问题。

### 图层数据结构

`denoise_run()` 构建 OptiX 图层：
- `OptixDenoiserGuideLayer` — 引导层（albedo + normal + flow）
- `OptixDenoiserLayer` — 图像层（input + output + previousOutput）
- 颜色格式统一使用 `OPTIX_PIXEL_FORMAT_FLOAT3`，光流使用 `OPTIX_PIXEL_FORMAT_FLOAT2`

## 关联文件

- `src/integrator/denoiser_gpu.h/.cpp` — GPU 降噪基类
- `src/integrator/denoiser.h/.cpp` — 降噪器基类与工厂方法
- `src/device/optix/device_impl.h` — OptiX 设备实现
- `src/device/optix/queue.h` — OptiX 队列（CUDA stream 封装）
