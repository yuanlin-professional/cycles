# pass_accessor_cpu.h / pass_accessor_cpu.cpp - CPU 渲染通道访问器

## 概述

本文件实现了 `PassAccessor` 的 CPU 侧子类 `PassAccessorCPU`，通过调用 CPU 内核函数在主机内存上执行渲染通道的读取和转换操作。它使用 TBB 并行化逐行处理，支持浮点和半精度 RGBA 两种输出格式。

## 类与结构体

### PassAccessorCPU

- **继承**: `PassAccessor`
- **功能**: 在 CPU 端执行渲染缓冲区到目标像素格式的转换，使用 `CPUKernels` 中的胶片转换内核函数。
- **关键成员**: 无额外成员（继承 `PassAccessor` 的构造函数）
- **关键方法**:
  - `run_get_pass_kernel_processor_float()` — 使用 TBB `parallel_for` 逐行调用浮点胶片转换内核
  - `run_get_pass_kernel_processor_half_rgba()` — 使用 TBB `parallel_for` 逐行调用半精度 RGBA 胶片转换内核
  - 各通道 `get_pass_*()` 虚函数实现 — 通过宏 `DEFINE_PASS_ACCESSOR` 统一生成

## 核心函数

### run_get_pass_kernel_processor_float()

并行处理浮点像素输出：
1. 计算窗口区域在缓冲区中的起始位置
2. 使用 `parallel_for` 对每一行并行调用 CPU 内核函数
3. 内核函数从渲染缓冲区读取一行像素并写入目标浮点数组

### run_get_pass_kernel_processor_half_rgba()

并行处理半精度 RGBA 像素输出：
1. 计算窗口起始位置和目标步幅
2. 使用 `parallel_for` 对每一行并行调用半精度内核函数
3. 支持自定义目标步幅（`destination.stride`）

### DEFINE_PASS_ACCESSOR 宏

通过宏展开为各通道生成统一的访问器实现：
```
get_pass_depth, get_pass_mist, get_pass_volume_majorant, get_pass_sample_count,
get_pass_float, get_pass_light_path, get_pass_shadow_catcher, get_pass_rgbe,
get_pass_float3, get_pass_motion, get_pass_cryptomatte,
get_pass_shadow_catcher_matte_with_shadow, get_pass_combined, get_pass_float4
```

每个实现：
1. 获取 `CPUKernels` 实例
2. 初始化 `KernelFilmConvert` 参数
3. 如果有浮点目标，调用 `run_get_pass_kernel_processor_float()`（对应 `kernels.film_convert_*`）
4. 如果有半精度目标，调用 `run_get_pass_kernel_processor_half_rgba()`（对应 `kernels.film_convert_half_rgba_*`）

## 依赖关系

- **内部头文件**:
  - `integrator/pass_accessor.h` — 基类
  - `device/cpu/kernel.h` — CPU 内核函数指针（`CPUKernels`）
  - `device/device.h` — `Device::get_cpu_kernels()` 静态方法
  - `session/buffers.h` — 渲染缓冲区
  - `util/tbb.h` — TBB 并行工具（`parallel_for`）
  - `kernel/types.h` — `KernelFilmConvert` 结构体
- **被引用**:
  - `integrator/denoiser_oidn.cpp` — OIDN CPU 降噪器中读取渲染通道
  - `integrator/path_trace_work_cpu.cpp` — CPU 路径追踪工作
  - `integrator/path_trace_tile.cpp` — 路径追踪瓦片处理

## 实现细节 / 关键算法

### TBB 并行化

所有内核调用都通过 `parallel_for(0, window_height, ...)` 对图像行进行并行处理。每一行的处理是独立的，因此可以高效利用多核 CPU。

### 窗口区域计算

渲染缓冲区可能大于实际渲染窗口（如视口导航期间的低分辨率渲染）。通过 `buffer_params.window_x/y/width/height` 定位实际数据区域，并用 `buffer_row_stride` 正确计算行间距。

### 双输出路径

每个通道的访问器同时支持两种输出格式：
- **浮点输出**（`destination.pixels`）: 用于最终渲染和数据传递
- **半精度 RGBA 输出**（`destination.pixels_half_rgba`）: 用于视口实时显示

两者互不排斥，可以同时输出。

## 关联文件

- `src/integrator/pass_accessor.h/.cpp` — 基类
- `src/integrator/pass_accessor_gpu.h/.cpp` — GPU 对应实现
- `src/device/cpu/kernel.h` — CPU 内核函数定义
