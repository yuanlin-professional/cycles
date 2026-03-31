# merge.h / merge.cpp - 多层 EXR 渲染图像合并

## 概述
本文件实现了 Cycles 渲染器的图像合并功能，用于将多个分开渲染的 OpenEXR 多层图像合并为单个输出文件。这在分布式渲染或多次渲染叠加采样时非常有用。合并器能够智能处理不同类型的通道：复制型通道（如深度）取第一个图像的值，累加型通道（如调试统计）直接求和，平均型通道（如颜色）按采样数加权平均。

## 类与结构体

### ImageMerger
- **功能**: 图像合并的顶层控制器，接收输入文件列表和输出路径，执行合并操作。
- **关键成员**:
  - `error` (`string`) — 运行失败后的错误信息
  - `input` (`vector<string>`) — 待合并的图像文件路径列表
  - `output` (`string`) — 输出文件路径
- **关键方法**:
  - `run()` — 执行完整合并流程：打开图像 -> 读取采样计数 -> 合并元数据和通道 -> 合并像素 -> 保存输出

### MergeImagePass（内部结构体）
- **功能**: 描述合并过程中的单个通道信息。
- **关键成员**:
  - `channel_name` (`string`) — 完整通道名
  - `name` (`string`) — 通道名
  - `format` (`TypeDesc`) — 文件中的通道格式
  - `op` (`MergeChannelOp`) — 合并操作类型
  - `offset` / `merge_offset` (`int`) — 在输入图像和合并图像中的通道偏移

### SampleCount（内部结构体）
- **功能**: 记录每层的采样计数信息。
- **关键成员**:
  - `total` (`int`) — 总采样数
  - `per_pixel` (`array<float>`) — 每像素的实际采样数缓冲区

### MergeImageLayer（内部结构体）
- **功能**: 描述合并过程中的渲染层。
- **关键成员**:
  - `name` (`string`) — 层名称
  - `passes` (`vector<MergeImagePass>`) — 该层的所有通道
  - `samples` (`int`) — 该层的采样数
  - `has_sample_pass` (`bool`) — 是否包含 "Debug Sample Count" 通道
  - `sample_pass_offset` (`int`) — 采样计数通道的偏移

### MergeImage（内部结构体）
- **功能**: 表示一个待合并的输入图像。
- **关键成员**:
  - `in` (`unique_ptr<ImageInput>`) — OIIO 文件句柄
  - `filepath` (`string`) — 文件路径
  - `layers` (`vector<MergeImageLayer>`) — 解析出的渲染层

## 枚举与常量

### MergeChannelOp
- `MERGE_CHANNEL_NOP` — 不做操作（重复通道时跳过）
- `MERGE_CHANNEL_COPY` — 复制（Depth、IndexMA、IndexOB、Crypto 通道，取第一个图像的值）
- `MERGE_CHANNEL_SUM` — 累加（Debug BVH、Debug Ray、Debug Render Time 通道）
- `MERGE_CHANNEL_AVERAGE` — 按采样数加权平均（大多数颜色通道的默认操作）
- `MERGE_CHANNEL_SAMPLES` — 采样计数通道的特殊处理（Debug Sample Count）

## 核心函数

### `parse_channel_operation()`
- **功能**: 根据通道名称确定合并操作类型。

### `parse_channels()`
- **功能**: 解析输入图像的通道名，按渲染层分组，确定每个通道的合并操作类型和采样数。支持多视图格式。

### `open_images()`
- **功能**: 打开所有输入图像，验证尺寸和格式一致性，解析通道信息。不支持深度图像（deep images）。

### `merge_channels_metadata()`
- **功能**: 合并所有图像的通道列表和元数据。基于第一个图像的规格，添加新通道、合并渲染时间统计、累计采样数。

### `merge_pixels()`
- **功能**: 执行像素级合并。根据通道操作类型分别处理：COPY（直接拷贝）、SUM（累加）、AVERAGE（按采样权重加权平均）、SAMPLES（输出归一化采样比例）。

### `read_layer_samples()`
- **功能**: 读取所有图像中每层的采样计数，支持从 "Debug Sample Count" 通道读取逐像素采样数或从元数据读取统一采样数。

### `save_output()`
- **功能**: 将合并结果保存到文件。使用临时文件写入再重命名的安全策略。

### `merge_render_time()` / `merge_layer_render_time()`
- **功能**: 合并渲染时间元数据，可选择累加或取平均。同步时间取平均，总时间和渲染时间取累加。

## 依赖关系
- **内部头文件**: `util/array.h`、`util/map.h`、`util/time.h`、`util/unique_ptr.h`、`util/string.h`、`util/vector.h`
- **外部库**: OpenImageIO（`imageio.h`、`filesystem.h`）
- **被引用**: `session/merge.cpp`（仅自身实现文件引用头文件）

## 实现细节 / 关键算法

### 加权平均合并算法
对于 `MERGE_CHANNEL_AVERAGE` 操作，每个像素的合并公式为：
```
output[pixel] = sum(input[pixel] * layer_samples[pixel] / total_samples[pixel])
```
其中 `layer_samples` 可以是逐像素值（来自 "Debug Sample Count" 通道）或统一值（来自元数据），`total_samples` 是所有输入图像对应位置的采样数之和。这确保了在自适应采样场景下不同区域按实际采样数正确加权。

### 通道映射策略
- 第一个图像中的所有通道被完整保留
- 后续图像中的同名通道合并到已有通道上
- 新通道被追加到输出规格中
- COPY 类型的通道在第二个及后续图像中降级为 NOP（不操作）

### 安全文件写入
输出先写入 `.merge-tmp-<unique>` 临时文件，成功后原子重命名为目标路径。失败时清理临时文件，不会损坏已有数据。

## 关联文件
- `session/denoising.h` — 降噪管线中也使用类似的通道解析和安全文件写入逻辑
- `util/time.h` — 提供 `time_human_readable_to_seconds()` 和 `time_human_readable_from_seconds()` 用于渲染时间元数据转换
- `scene/pass.h` — 渲染通道类型定义
