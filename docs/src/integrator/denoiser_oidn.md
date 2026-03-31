# denoiser_oidn.h / denoiser_oidn.cpp - OpenImageDenoise CPU 降噪器

## 概述

本文件实现了基于 OpenImageDenoise (OIDN) 库的 CPU 降噪器 `OIDNDenoiser`。它直接继承自 `Denoiser` 基类（非 `DenoiserGPU`），在 CPU 端执行完整的降噪流程。作为所有降噪方案的最终回退实现，当 GPU 降噪不可用时总会使用此实现。

## 类与结构体

### OIDNDenoiser

- **继承**: `Denoiser`
- **功能**: 基于 OpenImageDenoise 库的 CPU 降噪器，支持多通道降噪（color、albedo、normal），以及自适应采样的逐像素缩放。
- **关键成员**:
  - `mutex_` — 静态线程互斥锁，确保同时只有一个 OIDN 降噪任务执行（OIDN 自身已多线程化）
- **关键方法**:
  - `denoise_buffer()` — 执行 CPU 降噪的主入口：拷贝缓冲区到主机、读取引导通道、依次降噪三类通道
  - `get_device_type_mask()` — 返回 `DEVICE_MASK_CPU`

### OIDNPass（内部类）

- **功能**: 封装单个 OIDN 渲染通道的元信息和缓冲区引用。
- **关键成员**:
  - `name` — OIDN 图像名称（"color"、"albedo"、"normal"、"output"）
  - `type` / `mode` — 通道类型和模式
  - `offset` — 在渲染缓冲区中的偏移量
  - `need_scale` — 是否需要按采样数缩放（albedo/normal 通道需要）
  - `is_filtered` — 引导通道是否已经过预过滤
  - `scaled_buffer` — 缩放后数据的临时缓冲区

### OIDNDenoiseContext（内部类）

- **功能**: 管理一次完整 CPU 降噪操作的上下文，包含 OIDN 设备创建、滤波器配置、通道读取及后处理的全部逻辑。
- **关键方法**:
  - `need_denoising()` — 检查是否需要降噪（宽高非零）
  - `read_guiding_passes()` — 读取并预处理 albedo 和 normal 引导通道
  - `denoise_pass(pass_type)` — 对指定通道执行完整降噪流程
  - `filter_guiding_pass_if_needed()` — 对引导通道执行预过滤（ACCURATE 模式）
  - `set_quality()` — 根据用户设置配置 OIDN 质量级别（Fast/Balanced/High）
  - `postprocess_output()` — 对降噪输出进行缩放和 alpha 通道恢复
  - `set_fake_albedo_pass()` — 为不需要反照率的通道创建伪反照率缓冲区（值为 0.5）

## 核心函数

- `oidn_progress_monitor_function()` — OIDN 进度回调，检查用户是否请求取消
- `create_device_queue()` — 为渲染缓冲区所在设备创建 GPU 队列（若有），用于高效的数据传输
- `copy_render_buffers_from_device()` / `copy_render_buffers_to_device()` — 在设备与主机之间同步缓冲区数据

## 依赖关系

- **内部头文件**:
  - `integrator/denoiser.h` — 基类
  - `util/thread.h` — 线程互斥锁
  - `integrator/pass_accessor_cpu.h` — CPU 渲染通道访问器
  - `device/device.h`、`device/queue.h` — 设备接口
  - `session/buffers.h` — 渲染缓冲区
  - `util/openimagedenoise.h` — OIDN 支持检测
- **被引用**:
  - `integrator/denoiser.cpp` — 工厂方法中创建实例

## 实现细节 / 关键算法

### 降噪流水线

`denoise_buffer()` 的完整执行流程：

1. 获取线程锁（`thread_scoped_lock`），确保单一降噪实例
2. 从设备端拷贝渲染缓冲区到主机内存
3. 创建 `OIDNDenoiseContext`
4. 读取引导通道（albedo、normal）
5. 依次降噪三类通道：
   - `PASS_COMBINED`（合成通道，使用真实反照率）
   - `PASS_SHADOW_CATCHER_MATTE`（阴影捕捉遮罩，使用真实反照率）
   - `PASS_SHADOW_CATCHER`（阴影捕捉，使用伪反照率）
6. 将降噪结果拷回设备

### 通道降噪流程

每个通道的降噪（`denoise_pass()`）：
1. 创建 OIDN CPU 设备（`oidn::newDevice(CPU)`），禁用线程亲和性
2. 创建 "RT"（光线追踪）滤波器
3. 设置输入颜色、引导通道、输出图像
4. 配置 HDR 模式、质量等级、自定义权重
5. 按预过滤模式设置 `cleanAux` 标志
6. 对引导通道执行预过滤（ACCURATE 模式下）
7. 执行主滤波器
8. 后处理：按自适应采样的逐像素采样数缩放输出，恢复 alpha 通道

### 内存优化策略

引导通道的读取根据条件选择不同策略：
- **无需缩放时**: 直接引用渲染缓冲区中的数据（零拷贝）
- **允许就地修改时**: 直接在渲染缓冲区中缩放
- **需要保护原始数据时**: 分配临时缓冲区存储缩放后的数据

### 自定义权重

支持通过环境变量 `CYCLES_OIDN_CUSTOM_WEIGHTS` 加载自定义 OIDN 降噪权重文件。

## 关联文件

- `src/integrator/denoiser.h/.cpp` — 降噪器基类与工厂方法
- `src/integrator/pass_accessor_cpu.h/.cpp` — CPU 渲染通道读取
- `src/session/buffers.h` — `RenderBuffers` 定义
- `src/util/openimagedenoise.h` — OIDN 支持检测工具
