# path_trace_tile.h / path_trace_tile.cpp - 路径追踪图块数据访问接口

## 概述

`PathTraceTile` 实现了 `OutputDriver::Tile` 接口，为路径追踪器提供了对渲染图块像素数据的读写功能。它是 `PathTrace` 与外部输出驱动（`OutputDriver`）之间的桥梁，允许输出驱动按 pass 名称获取或设置像素数据。

该类在 `PathTrace::tile_buffer_write()` 流程中被创建，传递给输出驱动的回调函数，使得宿主应用可以按需获取各个渲染 pass 的像素结果。

## 类与结构体

### PathTraceTile

- **继承**: `OutputDriver::Tile`
- **功能**: 提供按 pass 名称读写图块像素数据的能力
- **关键成员**:
  - `path_trace_` — 所属 `PathTrace` 实例的引用
  - `copied_from_device_` — mutable 标志，记录是否已从设备拷贝数据（延迟拷贝优化）
- **关键方法**:
  - `get_pass_pixels(pass_name, num_channels, pixels)` — 按 pass 名称获取像素数据，写入到 `pixels` 缓冲区
  - `set_pass_pixels(pass_name, num_channels, pixels)` — 按 pass 名称设置像素数据（用于烘焙）

## 核心函数

- **`get_pass_pixels()`**: 核心的像素读取函数，流程如下：
  1. 按需从设备拷贝渲染缓冲区（延迟拷贝，仅首次调用时执行）
  2. 在缓冲区参数中查找指定名称的 pass
  3. 处理降噪模式回退：若请求降噪 pass 但无降噪结果，回退到噪声版本
  4. 处理显示 pass 的实际映射（`get_actual_display_pass`）
  5. 创建 `PassAccessorCPU` 并通过 `path_trace_.get_render_tile_pixels()` 获取像素

- **`set_pass_pixels()`**: 像素写入函数，用于烘焙场景中设置 pass 数据。查找 pass 后通过 `PassAccessorCPU` 调用 `path_trace_.set_render_tile_pixels()`。

## 依赖关系

- **内部头文件**:
  - `session/output_driver.h`（基类 `OutputDriver::Tile`）
- **实现文件额外依赖**:
  - `integrator/pass_accessor_cpu.h`, `integrator/path_trace.h`
  - `scene/pass.h`, `session/buffers.h`
- **被引用**: `integrator/path_trace.cpp`

## 实现细节 / 关键算法

1. **延迟设备拷贝**: `copied_from_device_` 标志确保从 GPU 到 CPU 的缓冲区拷贝仅在首次调用 `get_pass_pixels()` 时发生，后续对不同 pass 的读取复用已拷贝的数据，避免重复的设备同步开销。

2. **降噪回退策略**: 当请求降噪版本的 pass（`PassMode::DENOISED`）但降噪结果不可用时，自动回退到噪声版本的同类 pass。对于体积引导 pass 则视为始终有降噪结果。

3. **Shadow Catcher 近似处理**: 通过 `PassAccessInfo` 传递 `use_approximate_shadow_catcher` 和透明背景配置，影响 pass 访问器的像素转换行为。

4. **构造函数初始化**: 从 `PathTrace` 获取图块偏移、尺寸、渲染尺寸、图层和视图信息，传递给基类 `OutputDriver::Tile` 的构造函数。

## 关联文件

- `integrator/path_trace.h` — 上层路径追踪器，提供像素数据访问
- `session/output_driver.h` — 基类定义
- `integrator/pass_accessor_cpu.h` — CPU 侧 pass 数据访问器
