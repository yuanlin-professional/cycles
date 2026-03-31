# path_trace_work.h / path_trace_work.cpp - 路径追踪设备级工作抽象基类

## 概述

`PathTraceWork` 是路径追踪工作的抽象基类，定义了单个计算设备上需要执行的所有渲染操作接口。它负责管理设备级的渲染缓冲区，提供采样渲染、显示更新、缓冲区拷贝、自适应采样等纯虚接口，由 `PathTraceWorkCPU` 和 `PathTraceWorkGPU` 分别实现。

该类通过工厂方法 `create()` 根据设备类型自动选择合适的实现。每个 `PathTraceWork` 实例对应一个实际的渲染设备（非 MultiDevice），并管理该设备上的一个渲染缓冲区切片。

## 类与结构体

### PathTraceWork::RenderStatistics

- **功能**: 渲染统计信息
- **关键成员**: `occupancy` — 设备占用率（0.0~1.0）

### PathTraceWork

- **功能**: 单设备路径追踪工作的抽象基类
- **关键成员**:
  - `device_` — 实际渲染设备指针（非 MultiDevice）
  - `film_` — 胶片/成像平面，用于访问显示 pass 配置
  - `device_scene_` — 设备侧场景数据
  - `buffers_` — 渲染缓冲区，负责该设备的缓冲区切片
  - `effective_full_params_` / `effective_big_tile_params_` / `effective_buffer_params_` — 生效的缓冲区参数（可能受分辨率分割器影响）
  - `cancel_requested_flag_` — 取消请求标志指针
- **关键方法（纯虚接口）**:
  - `init_execution()` — 初始化设备队列以执行内核
  - `render_samples(statistics, start_sample, samples_num, sample_offset)` — 同步阻塞式渲染指定数量的采样
  - `copy_to_display(display, pass_mode, num_samples)` — 将渲染结果拷贝到显示纹理
  - `destroy_gpu_resources(display)` — 销毁 GPU 资源（如图形互操作对象）
  - `copy_render_buffers_from_device()` / `copy_render_buffers_to_device()` — 设备与主机间缓冲区同步
  - `zero_render_buffers()` — 清零渲染缓冲区
  - `adaptive_sampling_converge_filter_count_active(threshold, reset)` — 自适应采样收敛检测与过滤
  - `cryptomatte_postproces()` — Cryptomatte 后处理
  - `denoise_volume_guiding_buffers()` — 体积引导缓冲区降噪
- **关键方法（已实现）**:
  - `create(device, film, device_scene, cancel_flag)` — 工厂方法，按设备类型创建 CPU 或 GPU 实现
  - `set_effective_buffer_params(...)` — 设置生效的缓冲区参数
  - `has_multiple_works()` — 判断是否有多个 work 共享同一大图块
  - `copy_to_render_buffers()` / `copy_from_render_buffers()` — 在切片和完整缓冲区间拷贝数据
  - `copy_from_denoised_render_buffers()` — 仅拷贝降噪通道
  - `get_render_tile_pixels()` / `set_render_tile_pixels()` — 按偏移获取/设置像素
  - `get_display_pass_access_info()` — 获取显示 pass 的访问信息
  - `get_display_destination_template()` — 构建显示目标模板（考虑图块和切片偏移）

## 核心函数

- **`create()`**: 工厂方法。对 `DEVICE_CPU` 创建 `PathTraceWorkCPU`，对 `DEVICE_DUMMY` 返回 nullptr，其余设备创建 `PathTraceWorkGPU`。

- **`copy_to_render_buffers()`**: 先从设备拷贝缓冲区到主机，然后根据 Y 轴偏移计算在大图块中的位置，执行 `memcpy` 将工作缓冲区数据拷贝到目标渲染缓冲区的对应切片。

- **`copy_from_render_buffers()`**: 反向操作，从外部缓冲区的对应切片拷贝数据到工作缓冲区，然后上传到设备。

- **`get_display_destination_template()`**: 根据有效缓冲区参数和大图块参数计算在显示纹理中的偏移和步幅，构建 `PassAccessor::Destination` 模板。

## 依赖关系

- **内部头文件**:
  - `integrator/pass_accessor.h`, `scene/pass.h`, `session/buffers.h`, `util/unique_ptr.h`
- **实现文件额外依赖**:
  - `device/device.h`
  - `integrator/path_trace_display.h`, `integrator/path_trace_work_cpu.h`, `integrator/path_trace_work_gpu.h`
  - `scene/film.h`, `scene/scene.h`
  - `kernel/types.h`
- **被引用**: `integrator/path_trace.h`, `integrator/path_trace_work_cpu.h`, `integrator/path_trace_work_gpu.h`

## 实现细节 / 关键算法

1. **切片偏移计算**: 多设备渲染时，每个 `PathTraceWork` 负责大图块的一个水平切片。数据拷贝时通过 `effective_buffer_params_.full_y - effective_big_tile_params_.full_y` 计算 Y 轴偏移，将切片放入大图块的正确位置。

2. **多设备判断**: `has_multiple_works()` 通过比较有效缓冲区参数和大图块参数的尺寸及偏移来判断是否存在多个工作共享一个大图块。

3. **显示 pass 回退**: `get_display_pass_access_info()` 优先查找降噪版本的显示 pass，若不存在则回退到噪声版本。

4. **取消检查**: `is_cancel_requested()` 利用 x86 CPU 上标量读取的原子性，无需 atomic 操作即可在多线程环境中安全读取取消标志。

## 关联文件

- `integrator/path_trace_work_cpu.h` — CPU 设备实现
- `integrator/path_trace_work_gpu.h` — GPU 设备实现
- `integrator/path_trace.h` — 上层管理器，持有 `PathTraceWork` 列表
- `session/buffers.h` — `RenderBuffers` 定义
