# denoiser_oidn_gpu.h / denoiser_oidn_gpu.cpp - OpenImageDenoise GPU 降噪器

## 概述

本文件实现了基于 OpenImageDenoise (OIDN) 库的 GPU 降噪器 `OIDNDenoiserGPU`，继承自 `DenoiserGPU`。它利用 OIDN 2.x+ 的 GPU 后端，在 CUDA、HIP、SYCL (oneAPI) 或 Metal 设备上直接执行降噪运算，避免了 CPU 降噪的数据传输开销。编译需启用 `WITH_OPENIMAGEDENOISE` 宏。

## 类与结构体

### OIDNDenoiserGPU

- **继承**: `DenoiserGPU` -> `Denoiser`
- **功能**: 通过 OIDN C API 在 GPU 上执行降噪。管理 OIDN 设备和滤波器的生命周期，支持多种 GPU 后端。
- **关键成员**:
  - `oidn_device_` — OIDN 设备句柄
  - `oidn_filter_` — 主降噪滤波器
  - `albedo_filter_` — 反照率引导通道预过滤器
  - `normal_filter_` — 法线引导通道预过滤器
  - `is_configured_` / `configured_size_` — 配置状态和已配置的尺寸
  - `use_pass_albedo_` / `use_pass_normal_` — 当前使用的引导通道标志
  - `quality_` — 降噪质量等级
  - `custom_weights` — 自定义 OIDN 权重数据
  - `max_mem_` — 内存限制（初始 768MB），用于处理显存不足时的回退
- **关键方法**:
  - `is_device_supported()` — 静态方法，检测设备是否支持 OIDN GPU 降噪（通过 PCI ID 匹配或物理设备枚举）
  - `denoise_buffer()` — 委托给 `DenoiserGPU::denoise_buffer()`
  - `denoise_create_if_needed()` — 按需创建 OIDN 设备和滤波器（SYCL/Metal/CUDA/HIP）
  - `denoise_configure_if_needed()` — 检查并更新配置尺寸
  - `denoise_run()` — 执行实际降噪：设置颜色/引导/输出图像，提交并执行滤波器
  - `create_filter()` — 创建 OIDN "RT" 滤波器并设置质量级别
  - `commit_and_execute_filter()` — 提交并执行滤波器，含显存不足时的自动重试机制
  - `set_filter_pass()` — 设置滤波器图像参数，针对 Metal 使用 `oidnNewSharedBufferFromMetal`
  - `release_all_resources()` — 释放所有 OIDN 资源
  - `get_device_type_mask()` — 返回支持的设备类型掩码（ONEAPI/METAL/CUDA/OPTIX/HIP）

### ExecMode（枚举）

- `SYNC` — 同步执行
- `ASYNC` — 异步执行（用于引导通道预过滤以提高吞吐）

## 核心函数

- `oidn_device_type_to_string()` — 将 OIDN 设备类型转为可读字符串（调试用，OIDN < 2.3）

## 依赖关系

- **内部头文件**:
  - `integrator/denoiser_gpu.h` — GPU 降噪基类
  - `util/openimagedenoise.h` — OIDN 头文件包装
  - `device/device.h` — 设备接口
  - `device/oneapi/device_impl.h` — oneAPI 设备实现（获取 SYCL queue）
  - `device/queue.h` — 设备队列
  - `session/buffers.h` — 渲染缓冲区
- **被引用**:
  - `integrator/denoiser.cpp` — 工厂方法中条件创建（`WITH_OPENIMAGEDENOISE`）

## 实现细节 / 关键算法

### 设备匹配策略

`is_device_supported()` 通过两种方式匹配 OIDN 物理设备与 Cycles 设备：

- **OIDN >= 2.3**: 直接检查设备的 `denoisers` 能力标志
- **OIDN < 2.3**: 通过 PCI 总线地址匹配（`pciDomain:pciBus:pciDevice`），Metal 设备通过名称匹配

### OIDN 设备创建

根据 Cycles 设备类型调用对应的 OIDN 设备创建函数：
- oneAPI -> `oidnNewSYCLDevice()`
- Metal -> `oidnNewMetalDevice()`
- CUDA/OptiX -> `oidnNewCUDADevice()`
- HIP -> `oidnNewHIPDevice()`

### 显存不足重试

`commit_and_execute_filter()` 实现了自动内存降级机制：当 OIDN 返回 `OIDN_ERROR_OUT_OF_MEMORY` 时，将 `max_mem_` 减半后重试，直到内存限制降至 200MB 以下。

### 降噪执行流程

`denoise_run()` 的流程：
1. 设置颜色输入和输出图像（就地降噪，input = output）
2. 如果使用反照率引导，设置 albedo 图像并在 ACCURATE 模式下异步预过滤
3. 如果使用法线引导，设置 normal 图像并在 ACCURATE 模式下异步预过滤
4. 设置 `cleanAux` 标志（非 FAST 预过滤模式时为 true）
5. 同步提交并执行主滤波器

### Metal 特殊处理

Metal 设备需要通过 `oidnNewSharedBufferFromMetal()` 创建共享缓冲区来传递图像数据，而其他 GPU 后端直接使用 `oidnSetSharedFilterImage()` 传递设备指针。

## 关联文件

- `src/integrator/denoiser_gpu.h/.cpp` — GPU 降噪基类
- `src/integrator/denoiser.h/.cpp` — 降噪器基类与工厂方法
- `src/device/oneapi/device_impl.h` — oneAPI 设备（SYCL queue）
- `src/device/optix/device_impl.h` — CUDA/OptiX 设备
