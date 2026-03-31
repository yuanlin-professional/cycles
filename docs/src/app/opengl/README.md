# app/opengl - OpenGL 显示后端

## 概述

`src/app/opengl/` 模块为 Cycles 独立应用程序提供基于 OpenGL 的交互式视口显示后端。当编译启用 `WITH_CYCLES_STANDALONE_GUI` 选项时，该模块通过 SDL2 创建窗口并使用 OpenGL 实时显示渲染结果，支持键盘/鼠标交互操作（相机平移、旋转、弹跳次数调节等）。

该模块仅在独立应用程序的 GUI 模式下使用，后台渲染模式不加载此模块。

## 目录结构

| 文件名 | 类型 | 说明 |
|--------|------|------|
| `display_driver.h` | C++ 头文件 | `OpenGLDisplayDriver` 类声明，继承自 `DisplayDriver`，管理 OpenGL 纹理、PBO（Pixel Buffer Object）、顶点缓冲区、图形互操作（Graphics Interop）以及渲染/上传同步栅栏（GLsync） |
| `display_driver.cpp` | C++ 源文件 | `OpenGLDisplayDriver` 实现，包括纹理资源创建与管理、PBO 映射/解映射、纹理更新上传、顶点缓冲区更新、GPU 同步栅栏管理以及最终的 OpenGL 绘制流程（混合、纹理采样、三角扇绘制） |
| `shader.h` | C++ 头文件 | `OpenGLShader` 类声明，封装 GLSL 着色器程序的编译、链接及 uniform/attribute 位置缓存 |
| `shader.cpp` | C++ 源文件 | `OpenGLShader` 实现，包含内联的 GLSL 330 顶点/片段着色器源码。顶点着色器负责坐标归一化，片段着色器对纹理采样后应用硬编码的 Rec.709 gamma 校正（指数 0.45） |
| `window.h` | C++ 头文件 | 窗口管理接口声明，定义回调函数类型（初始化、退出、缩放、显示、键盘、鼠标移动）及上下文管理函数 |
| `window.cpp` | C++ 源文件 | 基于 SDL2 的窗口主循环实现，处理事件分发（键盘文本输入、鼠标按钮/移动、窗口缩放、退出），管理 OpenGL 上下文的线程安全切换（互斥锁），以及固定管线的视口设置和正交投影 |

## 核心类与数据结构

### OpenGLDisplayDriver

继承自 `session/display_driver.h` 中的 `DisplayDriver`，是渲染结果到屏幕显示的桥梁。

**关键职责：**

- **纹理管理** - 创建和维护 `GL_RGBA16F` 格式的 OpenGL 2D 纹理，通过 PBO 进行高效的像素数据传输
- **缓冲区映射** - 实现 `map_texture_buffer()`/`unmap_texture_buffer()`，将 PBO 映射为 `half4` 像素缓冲区供渲染线程写入
- **图形互操作** - 通过 `graphics_interop_get_device()`/`graphics_interop_update_buffer()` 支持 GPU 直接写入 PBO（零拷贝路径）
- **同步机制** - 使用 `glFenceSync` 创建上传同步栅栏（`gl_upload_sync_`）和渲染同步栅栏（`gl_render_sync_`），确保渲染线程与绘制线程不产生竞争
- **绘制流程** - `draw()` 方法执行完整的 OpenGL 渲染：启用混合、绑定着色器、上传纹理、更新顶点缓冲区、绘制三角扇

**内部纹理状态：**

- `gl_id` - OpenGL 纹理 ID
- `gl_pbo_id` - 像素缓冲对象 ID
- `need_update` - 新数据已写入 PBO，需上传到纹理
- `need_zero` - 纹理内容需清零
- `width`/`height` - 纹理尺寸
- `buffer_width`/`buffer_height` - PBO 缓冲区尺寸（按最终分辨率分配，避免频繁重建图形互操作对象）

### OpenGLShader

封装 GLSL 着色器程序的完整生命周期管理。

**着色器内容：**

- **顶点着色器**（GLSL 330）- 接收纹理坐标（`texCoord`）和位置（`pos`），将位置从像素坐标归一化到 NDC 空间
- **片段着色器**（GLSL 330）- 对 `image_texture` 采样并应用 pow(0.45) gamma 校正（硬编码 Rec.709，未来计划使用 OpenColorIO）

**关键方法：**

- `bind(width, height)` - 绑定着色器并设置 fullscreen uniform
- `create_shader_if_needed()` - 延迟编译，仅在首次使用时编译着色器
- `get_position_attrib_location()`/`get_tex_coord_attrib_location()` - 获取顶点属性位置

### Window 管理

`window.cpp` 中的全局 `Window V` 结构体维护窗口状态：

- **SDL 窗口/上下文** - `SDL_Window` 和 `SDL_GLContext`
- **回调系统** - 通过函数指针将窗口事件分发给 `cycles_standalone.cpp` 中的处理函数
- **线程安全** - `gl_context_mutex` 互斥锁确保 OpenGL 上下文在渲染线程和主线程间安全切换
- **事件循环** - `SDL_PollEvent` 处理输入事件，`SDL_WaitEventTimeout` 避免空转

**支持的交互操作（在 cycles_standalone.cpp 中实现）：**

- `h` - 显示/隐藏帮助信息
- `r` - 重置渲染
- `p` - 暂停/继续渲染
- `i` - 切换交互模式
- `W/A/S/D` - 相机移动
- `0/1/2/3` - 设置最大弹跳次数
- 鼠标左键拖拽 - 相机平移
- 鼠标右键拖拽 - 相机旋转
- `Esc` - 取消渲染
- `q` - 退出程序

## 依赖关系

### 上游依赖（本模块依赖）

- `session/display_driver.h` - `DisplayDriver` 基类和 `GraphicsInteropDevice` 定义
- `util/types.h` - 基础类型（`half4`、`float2`、`uint` 等）
- `util/log.h` - 日志系统
- `util/string.h` - 字符串工具
- `util/thread.h` - 线程互斥锁（`thread_mutex`）
- `util/version.h` - 版本信息
- SDL2 - 窗口创建与事件处理
- libepoxy - OpenGL 函数加载器

### 下游依赖（依赖本模块）

- `src/app/cycles_standalone.cpp` - 独立应用程序在 GUI 模式下使用 `OpenGLDisplayDriver` 和窗口管理函数

## 参见

- `src/app/` - 独立应用程序主模块
- `src/session/display_driver.h` - 显示驱动基类定义
- `src/session/output_driver.h` - 输出驱动基类定义（文件输出对应模块）
