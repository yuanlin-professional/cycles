# display_driver.h / display_driver.cpp - Hydra渲染代理的显示驱动实现

## 概述

本文件实现了 Cycles 渲染器在 Hydra渲染代理框架中的显示驱动层。`HdCyclesDisplayDriver` 继承自 Cycles 的 `DisplayDriver`，负责将渲染结果通过 OpenGL PBO（像素缓冲对象）传输到 Hydra 的渲染缓冲区纹理中，实现实时视口预览。该类管理 OpenGL 上下文的创建与切换（Windows 平台），处理 GPU 互操作以及渲染/上传之间的同步。

## 类与结构体

### HdCyclesDisplayDriver

- **继承**: `CCL_NS::DisplayDriver`
- **功能**: 将 Cycles 渲染器输出的 half4 格式像素数据通过 OpenGL PBO 上传到 HGI 纹理，供 Hydra 视口显示。
- **关键成员**:
  - `_renderParam` (`HdCyclesSession*`) — 渲染会话参数，用于获取 AOV 绑定和渲染缓冲区
  - `_hgi` (`Hgi*`) — Hydra 图形接口，用于创建和管理 GPU 纹理
  - `texture_` (`HgiTextureHandle`) — HGI 纹理句柄，存储渲染结果
  - `gl_pbo_id_` (`unsigned int`) — OpenGL PBO 标识符
  - `pbo_size_` (`int2`) — PBO 当前尺寸
  - `mutex_` (`thread_mutex`) — 保护 GL 上下文和资源的互斥锁
  - `need_update_` / `need_zero_` — 状态标志：是否需要更新纹理 / 是否需要清零
  - `gl_render_sync_` / `gl_upload_sync_` (`void*`) — OpenGL 栅栏同步对象
  - `hdc_` / `gl_context_` — Windows 平台的 DC 和 GL 上下文句柄
- **关键方法**:
  - `draw()` — 从 PBO 上传像素数据到 HGI 纹理（核心显示路径）
  - `update_begin()` / `update_end()` — 标记更新区间，管理 PBO 大小和 GL 同步
  - `map_texture_buffer()` / `unmap_texture_buffer()` — 映射/解映射 PBO 供 CPU 写入
  - `graphics_interop_*()` — GPU 互操作接口，用于 CUDA/HIP 等设备直接写入 PBO
  - `flush()` — 等待所有 GL 操作完成
  - `zero()` — 标记需要清零缓冲区
  - `gl_context_create()` / `gl_context_enable()` / `gl_context_disable()` / `gl_context_dispose()` — GL 上下文管理

## 核心函数

### `draw()`
- **功能**: 显示驱动的核心方法。创建或更新 HGI 纹理（`HgiFormatFloat16Vec4` 格式），通过 `glTexSubImage2D` 将 PBO 数据上传到纹理，并设置渲染栅栏同步。
- **流程**: 验证渲染缓冲区 -> 创建 GL 上下文 -> 创建/调整 HGI 纹理 -> 等待上传同步 -> PBO 到纹理拷贝 -> 设置渲染同步栅栏

### `gl_context_create()`
- **功能**: 在 Windows 平台上创建共享 GL 上下文。创建隐藏窗口，获取当前像素格式，创建新的 GL 上下文并通过 `wglShareLists` 与主上下文共享资源。同时创建 PBO。

## 依赖关系

- **内部头文件**:
  - `hydra/config.h` — 命名空间配置
  - `session/display_driver.h` — Cycles `DisplayDriver` 基类
  - `util/thread.h` — 线程互斥锁
  - `hydra/render_buffer.h` — `HdCyclesRenderBuffer`
  - `hydra/session.h` — `HdCyclesSession`
- **外部头文件**:
  - `pxr/imaging/hgi/hgi.h`, `pxr/imaging/hgi/texture.h` — Hydra 图形接口
  - `pxr/imaging/hgiGL/texture.h` — HGI OpenGL 纹理实现
  - `epoxy/gl.h` — OpenGL 函数加载
  - `Windows.h` — Windows 平台 API（仅 Windows）
- **被引用**:
  - `hydra/display_driver.cpp`, `hydra/render_pass.cpp`（通过 `HdCyclesDisplayDriver` 类名引用）

## 实现细节 / 关键算法

1. **双缓冲同步**: 使用两个 GL 栅栏同步对象（`gl_render_sync_` 和 `gl_upload_sync_`）实现渲染线程和显示线程之间的无锁同步。`update_begin()` 等待渲染完成，`draw()` 等待上传完成。

2. **PBO 动态调整**: 当渲染分辨率变化时，`update_begin()` 重新分配 PBO 大小，并清除 GPU 互操作缓冲区以触发重新初始化。

3. **Windows GL 上下文共享**: 创建独立的隐藏窗口和 GL 上下文，通过 `wglShareLists` 与主线程上下文共享 PBO 和纹理资源。非主线程激活该独立上下文，主线程则跳过上下文切换。

4. **CPU/GPU 双路径**: 支持 CPU 路径（`map_texture_buffer()` 直接映射 PBO 内存）和 GPU 互操作路径（`graphics_interop_update_buffer()` 提供 PBO 的 GL 标识符供 CUDA 等设备直接写入）。

5. **清零优化**: `need_zero_` 标志延迟到下次映射或互操作更新时执行清零，避免不必要的 GPU 操作。

## 关联文件

- `src/session/display_driver.h` — Cycles `DisplayDriver` 基类定义
- `src/hydra/render_buffer.h` — `HdCyclesRenderBuffer`（接收纹理资源）
- `src/hydra/render_pass.cpp` — 创建 `HdCyclesDisplayDriver` 实例
- `src/hydra/session.h` — `HdCyclesSession`（提供 AOV 绑定信息）
