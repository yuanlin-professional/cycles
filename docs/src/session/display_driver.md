# display_driver.h / display_driver.cpp - 显示驱动接口与图形互操作缓冲区

## 概述
本文件定义了 Cycles 渲染器与宿主应用程序之间的交互式渲染显示接口。`DisplayDriver` 是一个纯虚基类，宿主应用（如 Blender）通过实现该接口来将渲染结果实时显示到视口中。为了实现高效的 GPU 到显示的数据传输，文件还提供了 `GraphicsInteropDevice` 和 `GraphicsInteropBuffer` 类，支持 OpenGL、Vulkan、Metal 等图形 API 的零拷贝互操作机制。

## 类与结构体

### GraphicsInteropDevice
- **功能**: 描述用于图形互操作的显示设备信息，以便验证互操作是否与渲染设备兼容。
- **关键成员**:
  - `type` (`Type`) — 图形 API 类型枚举
  - `uuid` (`vector<uint8_t>`) — 设备唯一标识符
- **枚举**:
  - `Type`: `NONE`、`OPENGL`、`VULKAN`、`METAL`

### GraphicsInteropBuffer
- **功能**: 封装原生图形 API 像素缓冲区的句柄，支持渲染设备直接写入显示缓冲区而无需 CPU 中转。缓冲区格式为 `half4`（半精度浮点 RGBA）。
- **关键成员**:
  - `type_` (`GraphicsInteropDevice::Type`) — 句柄所属的图形 API 类型
  - `handle_` (`int64_t`) — 原生句柄（OpenGL: PBO ID; Vulkan/Windows: 不透明句柄; Vulkan/Unix: 文件描述符; Metal: MTLBuffer）
  - `own_handle_` (`bool`) — 是否拥有句柄的所有权
  - `size_` (`size_t`) — 缓冲区实际内存大小（必须 >= `width * height * sizeof(half4)`）
  - `need_zero_` (`bool`) — 在部分写入前是否需要清零整个缓冲区
- **关键方法**:
  - `assign(type, handle, size)` — 分配句柄，对 Vulkan 会接管所有权
  - `is_empty()` — 检查是否已分配句柄
  - `zero()` — 标记缓冲区需要清零
  - `clear()` — 清除句柄（Vulkan 句柄会被平台相关的 close 操作释放）
  - `get_type()` / `get_size()` — 获取类型和大小
  - `has_new_handle()` — 检查是否有新的句柄待接管
  - `take_handle()` — 接管句柄所有权
  - `take_zero()` — 接管清零标志

### DisplayDriver
- **功能**: 交互式渲染显示的抽象驱动接口。宿主应用实现此接口以接收渲染更新并在视口中绘制结果。
- **内部类 `Params`**:
  - `size` (`int2`) — 渲染分辨率（忽略渐进分辨率变化），纹理缓冲区应按此尺寸分配
  - `full_size` (`int2`) — 边界渲染时的完整分辨率
  - `full_offset` (`int2`) — 在完整渲染中的偏移
  - `modified()` — 检查参数是否变化
- **关键方法（纯虚函数）**:
  - `next_tile_begin()` — 通知开始处理下一个瓦片
  - `update_begin(params, width, height)` — 开始更新渲染纹理。`width`/`height` 是考虑渐进分辨率后的有效尺寸，可能小于 `params.size`。返回 `true` 表示可以继续更新
  - `update_end()` — 结束更新
  - `map_texture_buffer()` — 映射纹理缓冲区为 `half4*` 指针，供渲染设备写入
  - `unmap_texture_buffer()` — 解除纹理缓冲区映射
  - `zero()` — 清零显示缓冲区
  - `draw(params)` — 使用原生图形 API 绘制渲染结果。注意此方法可能与更新并行调用
- **关键方法（虚函数，可选覆盖）**:
  - `flush()` — 在结束渲染循环前刷新未完成的显示命令
  - `graphics_interop_get_device()` — 返回图形互操作设备信息
  - `graphics_interop_update_buffer()` — 更新图形互操作缓冲区内容
  - `graphics_interop_get_buffer()` — 获取图形互操作缓冲区引用
  - `graphics_interop_activate()` / `graphics_interop_deactivate()` — 激活/停用图形上下文（例如 CUDA 操作 OpenGL 对象时需要激活 OpenGL 上下文）
- **关键成员**:
  - `graphics_interop_buffer_` (`GraphicsInteropBuffer`) — 内置的图形互操作缓冲区

## 依赖关系
- **内部头文件**: `util/half.h`、`util/types.h`、`util/vector.h`
- **外部库**: Windows 平台使用 `util/windows.h`（`CloseHandle`），Unix 平台使用 `unistd.h`（`close`）
- **被引用**: `session/session.cpp`、`session/denoising.cpp`、`integrator/path_trace_display.h`、`integrator/path_trace_display.cpp`、`integrator/path_trace.cpp`、`integrator/denoiser.cpp`、`hydra/display_driver.h`、`app/opengl/display_driver.h` 等共 9 个文件

## 实现细节 / 关键算法
- **多线程更新协议**: `DisplayDriver` 的更新遵循特定的调用序列：`update_begin()` -> 并行的 `map_texture_buffer()`/`unmap_texture_buffer()` -> `update_end()`。多个渲染设备可以并行写入纹理缓冲区。`draw()` 可能与更新并行调用，实现者需自行处理互斥。
- **图形互操作机制**: 通过 `GraphicsInteropBuffer` 实现渲染设备与显示 GPU 之间的零拷贝数据传输。渲染设备可以直接写入图形 API 的像素缓冲区对象（PBO/VkBuffer/MTLBuffer），避免 CPU-GPU 之间的数据拷贝开销。
- **Vulkan 句柄生命周期**: Vulkan 平台的句柄需要特殊处理——`assign()` 时接管所有权，`clear()` 时在 Windows 上调用 `CloseHandle()`、Unix 上调用 `close()` 释放底层资源。
- **渐进分辨率支持**: `update_begin()` 接收的 `width`/`height` 可能小于 `params.size`，允许在不重新分配资源的情况下使用缓冲区的子集进行渐进渲染。

## 关联文件
- `session/session.h` — `Session::set_display_driver()` 设置显示驱动
- `integrator/path_trace_display.h` — `PathTraceDisplay` 使用 `DisplayDriver` 进行显示更新
- `session/output_driver.h` — 离线渲染的输出驱动接口（与 `DisplayDriver` 互补）
- `hydra/display_driver.h` — Hydra 渲染委托中的 `DisplayDriver` 实现
- `app/opengl/display_driver.h` — 独立应用中基于 OpenGL 的 `DisplayDriver` 实现
