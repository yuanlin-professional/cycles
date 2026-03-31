# path_trace_display.h / path_trace_display.cpp - 渲染结果交互式显示管理

## 概述

`PathTraceDisplay` 是对宿主应用（如 Blender）提供的 `DisplayDriver` 的线程安全封装，负责在路径追踪过程中高效地将渲染缓冲区内容更新到 GPU 侧纹理并进行绘制显示。它在 `DisplayDriver` 基础上增加了线程安全（互斥锁）、状态跟踪（纹理过期/映射状态）和错误检查功能。

该类支持三种纹理更新方式：CPU 缓冲区拷贝、纹理缓冲区映射（map/unmap）以及 GPU 图形互操作（graphics interop），以适配不同平台和性能需求。

## 类与结构体

### PathTraceDisplay

- **功能**: 管理渲染结果到 GPU 显示纹理的更新与绘制
- **关键成员**:
  - `driver_` — 宿主应用实现的 `DisplayDriver` 智能指针
  - `mutex_` — 线程互斥锁，保护参数读写
  - `params_` — 当前显示参数（全帧偏移、尺寸等）
  - `update_state_` — 更新过程状态（`is_active` 标记是否正在更新）
  - `texture_state_` — 纹理状态（`is_outdated` 过期标志、`size` 纹理尺寸）
  - `texture_buffer_state_` — 纹理缓冲区映射状态（`is_mapped` 标志）
- **关键方法**:
  - `reset(buffer_params, reset_rendering)` — 重置显示参数，标记纹理过期。非完全重置时调用 `driver_->next_tile_begin()`
  - `update_begin(texture_width, texture_height)` — 开始纹理更新（非阻塞），返回是否成功
  - `update_end()` — 结束纹理更新
  - `copy_pixels_to_texture(rgba_pixels, ...)` — 将 CPU 侧 half4 像素拷贝到纹理指定位置（通过 map/unmap 优化）
  - `map_texture_buffer()` / `unmap_texture_buffer()` — 映射/解映射纹理缓冲区以供 CPU 写入
  - `graphics_interop_get_device()` / `graphics_interop_get_buffer()` — 获取图形互操作设备和缓冲区
  - `graphics_interop_activate()` / `graphics_interop_deactivate()` — 激活/停用图形互操作
  - `zero()` — 清空纹理（填零），可与 `draw()` 并行但不能与 `update` 并行
  - `draw()` — 绘制当前纹理状态，返回是否绘制了最新数据
  - `flush()` — 刷新待处理的显示命令

## 核心函数

- **`copy_pixels_to_texture()`**: 先通过 `map_texture_buffer()` 获取映射指针，然后进行内存拷贝。支持全帧一次性 `memcpy` 和逐行拷贝两种路径。拷贝完成后自动 `unmap_texture_buffer()`。

- **`draw()`**: 在互斥锁保护下读取当前参数，然后非阻塞地调用 `driver_->draw(params)`。返回值基于 `texture_state_.is_outdated` 判断绘制的是否为最新内容。

- **`graphics_interop_get_buffer()`**: 获取图形互操作缓冲区前会检查纹理缓冲区未被映射且更新正在进行中，否则返回清空的缓冲区。调用 `mark_texture_updated()` 假定互操作会写入新数据。

## 依赖关系

- **内部头文件**:
  - `session/display_driver.h`
  - `util/half.h`, `util/thread.h`, `util/unique_ptr.h`
- **实现文件额外依赖**:
  - `session/buffers.h`
  - `util/log.h`
- **被引用**: `integrator/path_trace.cpp`, `integrator/path_trace_work.cpp`, `integrator/path_trace_work_cpu.cpp`, `integrator/path_trace_work_gpu.cpp`

## 实现细节 / 关键算法

1. **线程安全策略**: `mutex_` 仅保护 `params_` 和 `texture_state_.is_outdated` 的读写，更新过程本身（`update_begin/end` 之间）是非阻塞的，避免因子类持锁导致死锁。

2. **纹理状态追踪**: `is_outdated` 在 `reset()` 时设为 true，在 `mark_texture_updated()`（由 `copy_pixels_to_texture`/`unmap_texture_buffer`/`graphics_interop_get_buffer` 调用）时设为 false。`draw()` 返回 `!is_outdated` 表示是否绘制了最新内容。

3. **三种更新路径**:
   - CPU 拷贝: `copy_pixels_to_texture()` 内部通过 map/unmap 实现
   - 纹理映射: 直接 `map_texture_buffer()` 后写入再 `unmap_texture_buffer()`
   - 图形互操作: 通过 `graphics_interop_get_buffer()` 让 GPU 计算设备直接写入显示纹理

4. **互斥约束**: 纹理映射和图形互操作不能同时使用——`map` 期间不能调用 `graphics_interop_get_buffer()`，反之亦然。

## 关联文件

- `session/display_driver.h` — 宿主应用需要实现的显示驱动接口
- `integrator/path_trace.h` — 上层路径追踪器持有 `PathTraceDisplay`
- `integrator/path_trace_work.h` — 各设备工作通过此类更新显示
