# display_driver.h / display_driver.cpp - OpenGL 显示驱动实现

## 概述

本文件实现了基于 OpenGL 的显示驱动 `OpenGLDisplayDriver`，负责将 Cycles 渲染结果通过 OpenGL 纹理实时显示到屏幕上。它是 `DisplayDriver` 抽象基类的具体实现，属于独立应用（cycles_standalone）的 OpenGL 显示模块。设计上通过 PBO（Pixel Buffer Object）和 GPU 同步栅栏（fence sync）实现渲染线程与显示线程之间的高效数据传输，同时支持 GPU 图形互操作（Graphics Interop）以避免 CPU-GPU 之间的冗余数据拷贝。

## 类与结构体

### OpenGLDisplayDriver

- **继承**: `DisplayDriver`（定义于 `session/display_driver.h`）
- **功能**: 将路径追踪渲染输出的 half4 像素数据上传至 OpenGL 纹理，并在视口中绘制显示。管理纹理、PBO、顶点缓冲区等 GPU 资源的完整生命周期。
- **关键成员**:
  - `texture_` (匿名结构体) -- 管理渲染结果纹理及其关联的 PBO 的所有状态，包括：
    - `gl_id` -- OpenGL 纹理 ID
    - `gl_pbo_id` -- Pixel Buffer Object ID，用于高效像素写入
    - `need_update` -- 标记 PBO 中是否有新数据待上传到纹理
    - `need_zero` (`std::atomic<bool>`) -- 标记纹理是否需要清零，使用原子类型保证跨线程安全
    - `width`, `height` -- 纹理实际像素尺寸
    - `buffer_width`, `buffer_height` -- PBO 缓冲区尺寸（按最终渲染分辨率分配）
    - `creation_attempted`, `is_created` -- 防止 GPU 资源创建失败后反复重试
  - `display_shader_` (`OpenGLShader`) -- 显示用着色器程序，用于将纹理渲染到屏幕
  - `vertex_buffer_` (`uint`) -- 顶点缓冲区 ID，存储四边形三角扇的顶点与纹理坐标
  - `gl_render_sync_`, `gl_upload_sync_` (`void*`) -- OpenGL 同步栅栏对象（GLsync），分别用于绘制端和上传端的 GPU 同步
  - `zoom_` (`float2`) -- 缩放参数
  - `gl_context_enable_`, `gl_context_disable_` (`std::function`) -- OpenGL 上下文激活/停用回调，支持在渲染线程中独立于主线程操作 GL 上下文
- **关键方法**:
  - `OpenGLDisplayDriver(gl_context_enable, gl_context_disable)` -- 构造函数，接收 OpenGL 上下文启用/禁用的回调函数
  - `update_begin(params, texture_width, texture_height)` -- 开始更新渲染结果：启用 GL 上下文，等待渲染端同步栅栏，确保纹理资源就绪，按需调整纹理和 PBO 尺寸
  - `update_end()` -- 结束更新：创建上传端同步栅栏，刷新 GL 命令，禁用上下文
  - `map_texture_buffer()` -- 将 PBO 映射为 CPU 可写的 `half4*` 指针，如需清零则先执行 memset
  - `unmap_texture_buffer()` -- 解除 PBO 映射
  - `draw(params)` -- 核心绘制方法：等待上传同步，启用混合，绑定着色器和纹理，创建 VAO 配置顶点属性，以三角扇绘制四边形，最后创建渲染同步栅栏
  - `graphics_interop_get_device()` -- 返回 OPENGL 类型的图形互操作设备信息
  - `graphics_interop_update_buffer()` -- 更新图形互操作缓冲区，按需执行清零
  - `graphics_interop_activate()` / `graphics_interop_deactivate()` -- 激活/停用 GL 上下文以支持 CUDA 等外部 API 的互操作
  - `gl_texture_resources_ensure()` -- 延迟创建纹理和 PBO 资源，仅尝试一次
  - `gl_draw_resources_ensure()` -- 延迟创建绘制所需的顶点缓冲区，仅尝试一次
  - `gl_resources_destroy()` -- 销毁所有 GPU 资源（顶点缓冲区、PBO、纹理）
  - `texture_update_if_needed()` -- 当 PBO 中有新数据时，通过 `glTexSubImage2D` 上传到纹理
  - `vertex_buffer_update(params)` -- 根据渲染参数更新顶点缓冲区中的四个顶点（纹理坐标 + 位置坐标）
  - `set_zoom(zoom_x, zoom_y)` -- 设置显示缩放参数
  - `zero()` -- 标记纹理需要清零
  - `next_tile_begin()` -- 空实现，交互显示模式不使用分块

### texture_ (匿名结构体)

- **功能**: 封装 OpenGL 纹理及 PBO 的所有状态，作为 `OpenGLDisplayDriver` 的内嵌数据结构
- **设计特点**: PBO 按最终渲染分辨率（full_size）分配，避免分辨率除数变化时频繁重建图形互操作对象

## 枚举与常量

无独立定义的枚举或常量。纹理格式硬编码为 `GL_RGBA16F`（半精度浮点 RGBA）。

## 核心函数

所有核心功能均封装在 `OpenGLDisplayDriver` 类的方法中，无独立的模块级函数。

## 依赖关系

- **内部头文件**:
  - `app/opengl/shader.h` -- OpenGL 着色器封装
  - `session/display_driver.h` -- 显示驱动基类 `DisplayDriver`、`GraphicsInteropDevice`、`GraphicsInteropBuffer`
  - `util/log.h` -- 日志工具
- **外部库**:
  - `<SDL.h>` -- SDL 库（仅 .cpp 中引用，此处未直接使用 SDL API）
  - `<epoxy/gl.h>` -- OpenGL 函数加载库（提供所有 `gl*` 函数调用）
  - `<atomic>` -- 原子操作（`texture_.need_zero`）
  - `<functional>` -- 函数对象（上下文回调）
- **被引用**:
  - `src/app/cycles_standalone.cpp` -- 独立渲染应用创建 `OpenGLDisplayDriver` 实例

## 实现细节 / 关键算法

### 线程同步策略

更新（update）和绘制（draw）之间的互斥不在 Cycles 侧加锁，而是依赖 OpenGL 上下文的内部互斥锁（`DST.gl_context_lock`）。`update_begin()` 和 `draw()` 分别通过 `gl_context_enable_()` 获取上下文，从而避免锁反转（lock inversion）问题。

GPU 端使用 `glFenceSync` / `glWaitSync` 实现异步同步：
- `gl_upload_sync_`：在 `update_end()` 创建，在 `draw()` 中等待，确保上传完成后再绘制
- `gl_render_sync_`：在 `draw()` 末尾创建，在 `update_begin()` 中等待，确保绘制完成后再更新

### PBO 缓冲区策略

PBO 始终按 `params.full_size`（最终输出分辨率）分配，而非按当前有效渲染分辨率。这样在分辨率除数（resolution divider）不为 1 时，虽然会多传输一些数据，但避免了频繁重建图形互操作对象的高昂开销。

### 纹理插值

当纹理尺寸与显示参数的 `size` 不一致时（即分辨率除数不为 1），切换为最近邻插值（`GL_NEAREST`），否则使用线性插值（`GL_LINEAR`）。

### 资源延迟创建与单次尝试

`gl_texture_resources_ensure()` 和 `gl_draw_resources_ensure()` 均采用"尝试一次"模式：通过 `creation_attempted` 标志确保即使创建失败也不会在每帧重试，避免在 GPU 异常时造成性能问题。

### 顶点数据布局

顶点缓冲区包含 4 个顶点，每个顶点 4 个 float：前 2 个为纹理坐标 (u, v)，后 2 个为屏幕位置坐标 (x, y)。绘制采用 `GL_TRIANGLE_FAN` 模式。

## 关联文件

- `src/app/opengl/shader.h` / `shader.cpp` -- 显示着色器 `OpenGLShader`，被本类作为成员使用
- `src/app/opengl/window.h` / `window.cpp` -- SDL 窗口管理，提供 GL 上下文回调
- `src/session/display_driver.h` / `display_driver.cpp` -- 基类 `DisplayDriver` 及图形互操作基础设施
- `src/app/cycles_standalone.cpp` -- 创建并使用 `OpenGLDisplayDriver` 的独立应用入口
- `src/integrator/path_trace_display.h` / `path_trace_display.cpp` -- 路径追踪显示层，调用 `DisplayDriver` 接口
