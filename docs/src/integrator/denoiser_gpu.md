# denoiser_gpu.h / denoiser_gpu.cpp - GPU 降噪器基类

## 概述

本文件实现了 GPU 降噪的中间抽象层 `DenoiserGPU`，继承自 `Denoiser` 基类。它封装了 GPU 降噪的通用流程：缓冲区传输、引导通道（albedo/normal/flow）预处理、颜色通道读取与后处理，以及逐通道降噪的调度逻辑。具体的 GPU 降噪算法（OptiX 或 OIDN GPU）由子类实现。

## 类与结构体

### DenoiserGPU

- **继承**: `Denoiser`
- **功能**: GPU 降噪的通用基类，负责管理设备队列、缓冲区数据传输以及降噪流水线的编排。
- **关键成员**:
  - `denoiser_queue_` — GPU 设备队列（`unique_ptr<DeviceQueue>`），用于提交降噪内核
- **关键方法**:
  - `denoise_buffer(BufferParams, RenderBuffers, num_samples, allow_inplace)` — 公开接口，处理缓冲区在设备间的传输，然后调用内部降噪流程
  - `denoise_buffer(DenoiseTask)` — 内部降噪入口，依次执行：确保降噪器就绪 -> 预处理引导通道 -> 降噪各渲染通道
  - `denoise_ensure()` — 确保 GPU 降噪器已创建并配置
  - `denoise_create_if_needed()` — 纯虚函数，由子类创建特定的 GPU 降噪描述符
  - `denoise_configure_if_needed()` — 纯虚函数，由子类配置降噪描述符
  - `denoise_run()` — 纯虚函数，由子类执行实际降噪算法
  - `denoise_pass()` — 对单个渲染通道执行完整的降噪流程
  - `denoise_color_read()` — 使用 `PassAccessorGPU` 读取带噪声的颜色输入
  - `denoise_filter_color_preprocess()` — 颜色预处理（仅 OptiX 需要值截断）
  - `denoise_filter_color_postprocess()` — 颜色后处理（缩放至采样数匹配）
  - `denoise_filter_guiding_preprocess()` — 预处理引导通道（albedo、normal、flow）
  - `denoise_filter_guiding_set_fake_albedo()` — 为不需要反照率的通道设置伪反照率值（0.5）

### DenoiseTask

- **功能**: 降噪任务参数的封装类，聚合所有降噪所需参数。
- **关键成员**:
  - `params` — 降噪参数
  - `num_samples` — 采样数
  - `render_buffers` — 渲染缓冲区指针
  - `buffer_params` — 缓冲区参数
  - `allow_inplace_modification` — 是否允许就地修改输入通道（降低内存占用但会使输入通道失效）

### DenoisePass

- **功能**: 描述单个需要降噪的渲染通道。
- **关键成员**:
  - `type` — 通道类型（`PassType`）
  - `noisy_offset` / `denoised_offset` — 带噪声/降噪后数据在缓冲区中的偏移量
  - `num_components` — 分量数（如 RGB 为 3）
  - `use_compositing` — 是否需要合成处理
  - `use_denoising_albedo` — 是否使用反照率引导

### DenoiseContext

- **功能**: 降噪上下文，持有一次降噪操作所需的全部设备端状态。
- **关键成员**:
  - `guiding_buffer` — 引导通道的设备端存储
  - `guiding_params` — 引导通道参数（设备指针、各通道偏移、步幅）
  - `prev_output` — 前一帧输出（用于时序稳定降噪）
  - `use_pass_albedo` / `use_pass_normal` / `use_pass_motion` — 引导通道启用标志
  - `albedo_replaced_with_fake` — 标记是否已用伪反照率替换真实反照率

## 核心函数

降噪流程由 `denoise_buffer(DenoiseTask)` 驱动：

1. 创建 `DenoiseContext`，初始化引导通道存储
2. `denoise_ensure()` — 创建并配置 GPU 降噪器
3. `denoise_filter_guiding_preprocess()` — 通过 `DEVICE_KERNEL_FILTER_GUIDING_PREPROCESS` 内核预处理引导通道
4. `denoise_pass(PASS_COMBINED)` — 降噪合成通道
5. `denoise_pass(PASS_SHADOW_CATCHER_MATTE)` — 降噪阴影捕捉遮罩通道
6. `denoise_pass(PASS_SHADOW_CATCHER)` — 降噪阴影捕捉通道（使用伪反照率）

## 依赖关系

- **内部头文件**:
  - `integrator/denoiser.h` — 基类
  - `session/buffers.h` — 渲染缓冲区
  - `device/denoise.h`、`device/device.h`、`device/memory.h`、`device/queue.h` — 设备层接口
  - `integrator/pass_accessor_gpu.h` — GPU 渲染通道访问器
- **被引用**:
  - `integrator/denoiser_oidn_gpu.h`
  - `integrator/denoiser_optix.h`

## 实现细节 / 关键算法

### 跨设备缓冲区处理

当降噪设备与渲染缓冲区所在设备不同时，`denoise_buffer()` 会：
1. 将渲染缓冲区从渲染设备拷贝至主机内存
2. 在降噪设备上创建临时本地缓冲区
3. 执行降噪
4. 将降噪结果拷回原始渲染缓冲区

### 引导通道内存优化

`DenoiseContext` 构造函数根据 `allow_inplace_modification` 标志选择两种策略：
- **就地修改**: 直接在渲染缓冲区上操作，零额外内存开销
- **独立缓冲**: 分配独立的引导通道缓冲区，保护原始数据不被破坏

### 伪反照率机制

对于不需要反照率的渲染通道（如 `PASS_SHADOW_CATCHER`），需要用 (0.5, 0.5, 0.5) 替换真实反照率。一旦替换，后续需要真实反照率的通道将无法再降噪，因此降噪顺序严格要求先处理需要反照率的通道。

## 关联文件

- `src/integrator/denoiser.h/.cpp` — 降噪器基类
- `src/integrator/denoiser_oidn_gpu.h/.cpp` — OIDN GPU 降噪子类
- `src/integrator/denoiser_optix.h/.cpp` — OptiX 降噪子类
- `src/integrator/pass_accessor_gpu.h/.cpp` — GPU 渲染通道读取
