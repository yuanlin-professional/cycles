# denoising.h / denoising.cpp - 独立降噪管线与图像帧处理

## 概述
本文件实现了 Cycles 渲染器的独立降噪管线（Denoiser Pipeline），用于对已渲染的图像文件进行离线降噪处理。与集成在渲染会话中的实时降噪不同，此模块从磁盘读取多层 EXR 图像文件，提取降噪所需的通道数据（合成图像、法线、反照率、运动向量），在设备上执行降噪操作，并将结果写回文件。支持多帧时序稳定降噪。

## 类与结构体

### DenoiserPipeline
- **功能**: 降噪管线的顶层控制器，管理设备创建、降噪器初始化，并按帧顺序执行降噪任务。
- **关键成员**:
  - `error` (`string`) — 运行失败后的错误信息
  - `input` (`vector<string>`) — 输入帧文件路径列表（按顺序排列）
  - `output` (`vector<string>`) — 输出帧文件路径列表（空条目将被跳过）
  - `device` (`unique_ptr<Device>`) — 降噪计算设备
  - `cpu_device` (`unique_ptr<Device>`) — CPU 设备（作为降噪器的备用设备）
  - `denoiser` (`std::unique_ptr<Denoiser>`) — 降噪器实例
  - `stats` (`Stats`) / `profiler` (`Profiler`) — 统计与性能分析
- **关键方法**:
  - `run()` — 执行整个降噪流程：遍历所有帧，依次加载、降噪、保存
  - 构造函数 — 初始化任务调度器、创建设备、加载降噪内核

### DenoiseImageLayer
- **功能**: 表示图像中一个渲染层的通道映射信息，检测并配置降噪所需的完整通道集。
- **关键成员**:
  - `name` (`string`) — 层名称
  - `channels` (`vector<string>`) — 层内所有通道名称
  - `layer_to_image_channel` (`vector<int>`) — 层通道到图像文件通道的映射
  - `samples` (`int`) — 该层的采样数
  - `input_to_image_channel` (`vector<int>`) — 设备输入通道到图像通道的映射
  - `output_to_image_channel` (`vector<int>`) — 处理输出通道到图像通道的映射
  - `previous_output_to_image_channel` (`vector<int>`) — 前一帧输出通道的映射
- **关键方法**:
  - `detect_denoising_channels()` — 检测层是否包含完整的降噪通道集（Combined RGB、Denoising Normal XYZ、Denoising Albedo RGB、Vector XYZW），并设置输入/输出映射
  - `match_channels()` — 将相邻帧的通道与当前帧匹配，用于时序降噪

### DenoiseImage
- **功能**: 管理降噪图像数据的加载、像素读取与输出保存。
- **关键成员**:
  - `width` / `height` / `num_channels` (`int`) — 图像尺寸与通道数
  - `samples` (`int`) — 采样数
  - `pixels` (`array<float>`) — 交错存储的像素缓冲区
  - `in_spec` (`ImageSpec`) — 输入图像规格
  - `in_previous` (`unique_ptr<ImageInput>`) — 前一帧图像输入句柄
  - `layers` (`vector<DenoiseImageLayer>`) — 检测到的可降噪渲染层
- **关键方法**:
  - `load(filepath)` — 打开输入图像文件，解析通道，读取所有像素数据
  - `load_previous(filepath)` — 加载前一帧用于时序降噪
  - `read_pixels(layer, params, input_pixels)` — 将像素数据按通道映射重排并拷贝到设备输入缓冲区
  - `read_previous_pixels(layer, params, input_pixels)` — 读取前一帧像素到设备缓冲区
  - `save_output(filepath)` — 将降噪结果保存到文件（先写临时文件再重命名，确保安全）
  - `parse_channels(in_spec)` — 解析图像通道名，按渲染层分组，识别可降噪层

### DenoiseTask
- **功能**: 单帧降噪任务，封装加载、执行、保存的完整流程。
- **关键成员**:
  - `denoiser` (`DenoiserPipeline*`) — 所属管线
  - `device` (`Device*`) — 计算设备
  - `frame` (`int`) — 当前帧号
  - `image` (`DenoiseImage`) — 图像数据
  - `current_layer` (`int`) — 当前处理的渲染层索引
  - `buffers` (`RenderBuffers`) — 设备端渲染缓冲区
- **关键方法**:
  - `load()` — 加载帧图像、配置缓冲区通道（Combined + Albedo + Normal + Motion + Previous + Denoised Combined）、分配设备缓冲区
  - `exec()` — 对每个渲染层执行降噪，将降噪结果写回图像像素缓冲区
  - `save()` — 保存降噪结果到输出文件
  - `free()` — 释放图像和缓冲区资源
  - `load_input_pixels(layer)` — 加载指定层的像素数据到设备缓冲区

## 枚举与常量
- `INPUT_NUM_CHANNELS = 13` — 输入通道总数
- `INPUT_NOISY_IMAGE = 0` — 含噪声合成图像的起始通道
- `INPUT_DENOISING_NORMAL = 3` — 降噪法线的起始通道
- `INPUT_DENOISING_ALBEDO = 6` — 降噪反照率的起始通道
- `INPUT_MOTION = 9` — 运动向量的起始通道
- `OUTPUT_NUM_CHANNELS = 3` — 输出通道数（RGB）

## 核心函数
- `split_last_dot()` — 在最后一个点处分割字符串（用于解析通道名）
- `parse_channel_name()` — 解析 Blender 生成的通道名格式（`RenderLayer.Pass.View.Channel` 或 `RenderLayer.Pass.Channel`）
- `fill_mapping()` — 构建通道名称与位置的映射关系
- `input_channels()` / `output_channels()` — 返回降噪输入/输出的通道映射列表
- `add_pass()` — 辅助函数，向通道列表添加新通道

## 依赖关系
- **内部头文件**: `device/device.h`、`device/cpu/device.h`、`integrator/denoiser.h`、`session/buffers.h`、`session/display_driver.h`、`util/string.h`、`util/unique_ptr.h`、`util/vector.h`、`util/map.h`、`util/task.h`
- **外部库**: OpenImageIO（`imageio.h`、`filesystem.h`）
- **被引用**: `session/denoising.cpp`（仅自身实现文件引用头文件）

## 实现细节 / 关键算法
- **通道解析与映射**: 输入图像的通道名遵循 Blender 的命名规范。解析时按渲染层分组，然后检测每层是否包含完整的降噪通道集（Combined、Normal、Albedo、Motion）。通道映射允许在图像文件通道顺序和设备端缓冲区布局之间进行转换。
- **时序降噪**: 第一帧之后的帧会使用前一帧的降噪输出作为额外输入（`PASS_DENOISING_PREVIOUS`），通过设置 `params.temporally_stable = true` 启用时序稳定性。
- **安全文件写入**: 降噪结果先写入临时文件（`.denoise-tmp-*`），成功后重命名为目标文件，失败时删除临时文件，避免损坏原始数据。
- **设备初始化**: `DenoiserPipeline` 构造时创建主设备（可为 GPU）和 CPU 备用设备，使用 `Denoiser::create()` 工厂方法创建降噪器实例。

## 关联文件
- `integrator/denoiser.h` — `Denoiser` 基类，提供 `denoise_buffer()` 等降噪接口
- `session/buffers.h` — `RenderBuffers` 和 `BufferParams`，管理设备端数据
- `scene/pass.h` — 渲染通道类型定义
- `device/device.h` — 设备抽象层
