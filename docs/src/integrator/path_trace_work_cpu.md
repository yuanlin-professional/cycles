# path_trace_work_cpu.h / path_trace_work_cpu.cpp - CPU 设备路径追踪工作实现

## 概述

`PathTraceWorkCPU` 是 `PathTraceWork` 的 CPU 设备实现，采用逐像素调度方式，通过 TBB（Threading Building Blocks）并行框架在 CPU 多线程上执行路径追踪。它使用 megakernel（巨核）模式，每个线程独立处理一个像素的完整路径追踪流程。

该实现特别针对 CPU 架构优化，包括每线程独立的内核全局变量（`ThreadKernelGlobalsCPU`）、TBB task arena 线程池管理，以及条件编译的路径引导（path guiding）支持。

## 类与结构体

### PathTraceWorkCPU

- **继承**: `PathTraceWork`
- **功能**: 在 CPU 设备上以逐像素并行方式执行路径追踪
- **关键成员**:
  - `kernels_` — CPU 内核函数集合（`CPUKernels`）的引用
  - `kernel_thread_globals_` — 每线程独立的内核全局变量数组（`vector<ThreadKernelGlobalsCPU>`），用于线程解耦
- **关键方法**:
  - `init_execution()` — 缓存每线程的内核全局变量
  - `render_samples(statistics, start_sample, samples_num, sample_offset)` — TBB 并行逐像素渲染
  - `render_samples_full_pipeline(kernel_globals, work_tile, samples_num)` — 单像素的完整路径追踪管线
  - `copy_to_display(display, pass_mode, num_samples)` — 通过纹理映射将渲染结果拷贝到显示
  - `adaptive_sampling_converge_filter_count_active(threshold, reset)` — 自适应采样收敛检测与过滤
  - `cryptomatte_postproces()` — Cryptomatte 后处理内核调用
  - `denoise_volume_guiding_buffers()` — 体积散射概率引导缓冲区降噪（X/Y 方向两次过滤）
  - `guiding_init_kernel_globals(guiding_field, sample_data_storage, train)` — 初始化每线程的路径引导数据（条件编译）
  - `guiding_push_sample_data_to_global_storage(kg, state, render_buffer)` — 将路径引导训练数据推送到全局存储（条件编译）

## 核心函数

- **`render_samples()`**: CPU 渲染的入口函数，流程如下：
  1. 计算总像素数 `total_pixels_num = width * height`
  2. 创建与设备线程数匹配的 `tbb::task_arena`
  3. 在 arena 中以 `parallel_for` 并行遍历所有像素
  4. 每个像素构造 1x1 的 `KernelWorkTile`，获取当前线程的 `ThreadKernelGlobalsCPU`
  5. 调用 `render_samples_full_pipeline()` 完成该像素的所有采样
  6. 如开启 profiler，统计每线程性能数据

- **`render_samples_full_pipeline()`**: 单像素的完整 megakernel 管线：
  1. 初始化两个 `IntegratorStateCPU`（主路径和 shadow catcher 路径）
  2. 遍历每个采样：
     - 根据是否烘焙选择 `integrator_init_from_bake` 或 `integrator_init_from_camera`
     - 路径引导模式下调用 `integrator_megakernel` 后推送训练数据
     - 非引导模式直接调用 `integrator_megakernel`
     - Shadow catcher 路径在引导模式下禁用训练
  3. 记录渲染时间到 pass 缓冲区

- **`adaptive_sampling_converge_filter_count_active()`**: 自适应采样的收敛检测与过滤：
  1. 逐行并行执行收敛检查（`adaptive_sampling_convergence_check`）和 X 方向过滤
  2. 统计活跃像素数
  3. 若有活跃像素，再逐列并行执行 Y 方向过滤

- **`copy_to_display()`**: 通过 `map_texture_buffer()` 获取显示纹理的映射指针，使用 `PassAccessorCPU` 在 TBB arena 中并行转换像素格式并写入。

## 依赖关系

- **内部头文件**:
  - `kernel/device/cpu/globals.h`, `kernel/integrator/state.h`
  - `device/queue.h`
  - `integrator/path_trace_work.h`
  - `util/vector.h`
- **实现文件额外依赖**:
  - `device/cpu/kernel.h`, `device/device.h`
  - `kernel/integrator/path_state.h`
  - `integrator/pass_accessor_cpu.h`, `integrator/path_trace_display.h`
  - `scene/scene.h`, `session/buffers.h`
  - `util/tbb.h`, `util/time.h`
- **被引用**: `integrator/path_trace_work.cpp`（在工厂方法中创建）

## 实现细节 / 关键算法

1. **Megakernel 模式**: CPU 实现使用 megakernel（`integrator_megakernel`），即单个函数完成一条路径的全部弹射，而非 GPU 上的波前（wavefront）分步内核。这种方式更适合 CPU 的缓存和分支预测特性。

2. **TBB Arena**: 通过 `local_tbb_arena_create()` 创建线程数与 CPU 设备线程数匹配的 TBB task arena，确保 CPU+GPU 混合渲染时不会过度占用 CPU 线程。

3. **线程局部内核全局变量**: `kernel_thread_globals_` 为每个 TBB 线程维护独立的 `ThreadKernelGlobalsCPU`，通过 `tbb::this_task_arena::current_thread_index()` 索引，避免线程间共享状态冲突。

4. **路径引导训练**: 条件编译 `WITH_PATH_GUIDING` 下，每条路径的采样结果通过 `PathSegmentStorage` 收集，路径结束后经 `PrepareSamples()` 转换为辐射采样并推送到全局 `SampleStorage`。Shadow catcher 路径不参与训练。

5. **体积引导降噪**: `denoise_volume_guiding_buffers()` 执行两遍过滤：先对每个像素在 X 方向过滤（2D blocked_range 并行），再在 Y 方向过滤（按列并行，内核内部串行执行以避免中间缓冲区）。

## 关联文件

- `integrator/path_trace_work.h` — 基类定义
- `integrator/path_trace_work_gpu.h` — GPU 对应实现
- `device/cpu/kernel.h` — CPU 内核函数集合
- `kernel/device/cpu/globals.h` — `ThreadKernelGlobalsCPU` 定义
