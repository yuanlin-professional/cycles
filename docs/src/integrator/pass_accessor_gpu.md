# pass_accessor_gpu.h / pass_accessor_gpu.cpp - GPU 渲染通道访问器

## 概述

本文件实现了 `PassAccessor` 的 GPU 侧子类 `PassAccessorGPU`，通过设备队列将胶片转换内核提交到 GPU 执行，实现渲染缓冲区到目标像素格式的高效转换。支持浮点和半精度 RGBA 两种输出格式，所有操作在 GPU 上异步执行后同步等待完成。

## 类与结构体

### PassAccessorGPU

- **继承**: `PassAccessor`
- **功能**: 在 GPU 端执行渲染缓冲区到目标像素格式的转换，通过 `DeviceQueue` 提交设备内核。
- **关键成员**:
  - `queue_` — GPU 设备队列指针（`DeviceQueue*`），用于提交内核
- **关键方法**:
  - `PassAccessorGPU(queue, pass_access_info, exposure, num_samples)` — 构造函数，额外接收 GPU 队列
  - `run_film_convert_kernels()` — 核心执行函数，初始化参数、计算工作尺寸，向 GPU 提交内核
  - 各通道 `get_pass_*()` 虚函数实现 — 通过宏 `DEFINE_PASS_ACCESSOR` 统一生成

## 核心函数

### run_film_convert_kernels()

GPU 胶片转换的统一执行入口：

1. 初始化 `KernelFilmConvert` 参数结构体
2. 计算工作尺寸（`window_width * window_height`）
3. 计算缓冲区偏移量和目标步幅
4. 初始化设备执行环境（`queue_->init_execution()`）
5. 如果有浮点设备目标（`d_pixels`），提交对应的设备内核
6. 如果有半精度设备目标（`d_pixels_half_rgba`），提交 `kernel + 1` 的半精度内核
7. 同步等待所有操作完成（`queue_->synchronize()`）

### DEFINE_PASS_ACCESSOR 宏

通过宏展开为各通道生成统一的访问器实现：
```
get_pass_depth -> DEVICE_KERNEL_FILM_CONVERT_DEPTH
get_pass_mist -> DEVICE_KERNEL_FILM_CONVERT_MIST
get_pass_volume_majorant -> DEVICE_KERNEL_FILM_CONVERT_VOLUME_MAJORANT
get_pass_sample_count -> DEVICE_KERNEL_FILM_CONVERT_SAMPLE_COUNT
get_pass_float -> DEVICE_KERNEL_FILM_CONVERT_FLOAT
get_pass_light_path -> DEVICE_KERNEL_FILM_CONVERT_LIGHT_PATH
get_pass_rgbe -> DEVICE_KERNEL_FILM_CONVERT_RGBE
get_pass_float3 -> DEVICE_KERNEL_FILM_CONVERT_FLOAT3
get_pass_motion -> DEVICE_KERNEL_FILM_CONVERT_MOTION
get_pass_cryptomatte -> DEVICE_KERNEL_FILM_CONVERT_CRYPTOMATTE
get_pass_shadow_catcher -> DEVICE_KERNEL_FILM_CONVERT_SHADOW_CATCHER
get_pass_shadow_catcher_matte_with_shadow -> DEVICE_KERNEL_FILM_CONVERT_SHADOW_CATCHER_MATTE_WITH_SHADOW
get_pass_combined -> DEVICE_KERNEL_FILM_CONVERT_COMBINED
get_pass_float4 -> DEVICE_KERNEL_FILM_CONVERT_FLOAT4
```

## 依赖关系

- **内部头文件**:
  - `integrator/pass_accessor.h` — 基类
  - `kernel/types.h` — `KernelFilmConvert`、`DeviceKernel` 等定义
  - `device/queue.h` — GPU 设备队列
  - `session/buffers.h` — 渲染缓冲区
- **被引用**:
  - `integrator/denoiser_gpu.cpp` — GPU 降噪器中读取渲染通道
  - `integrator/denoiser_optix.cpp` — OptiX 降噪器
  - `integrator/path_trace_work_gpu.cpp` — GPU 路径追踪工作

## 实现细节 / 关键算法

### 内核编号约定

半精度 RGBA 版本的内核编号是浮点版本的 `kernel + 1`，这是通过 `DeviceKernel` 枚举的排列约定实现的。这样只需一个基础内核枚举值即可同时处理两种输出格式。

### 设备指针传递

所有数据通过设备指针（`device_ptr`）传递给内核，包括：
- 目标像素缓冲区（`d_pixels` 或 `d_pixels_half_rgba`）
- 渲染缓冲区（`render_buffers->buffer.device_pointer`）
- 工作尺寸和偏移量等标量参数

### 缓冲区偏移计算

偏移量计算考虑了窗口位置和通道步幅：
```
offset = window_x * pass_stride + window_y * stride * pass_stride
```

这确保了在部分缓冲区渲染（如视口导航）时正确定位数据。

## 关联文件

- `src/integrator/pass_accessor.h/.cpp` — 基类
- `src/integrator/pass_accessor_cpu.h/.cpp` — CPU 对应实现
- `src/device/queue.h` — `DeviceQueue` 接口定义
