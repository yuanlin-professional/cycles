# window.h / window.cpp - SDL 窗口管理与事件循环

## 概述

本文件提供了基于 SDL 的 OpenGL 窗口创建、事件处理和主循环功能，为 Cycles 独立应用（cycles_standalone）提供简化的窗口管理接口。设计目标是将窗口系统的样板代码降到最低，通过回调函数机制让调用者专注于渲染逻辑。文件采用全局单例模式管理窗口状态，并通过互斥锁实现 OpenGL 上下文的线程安全访问。

## 类与结构体

### Window (文件作用域结构体)

- **继承**: 无
- **功能**: 存储窗口的完整运行时状态，包括回调函数、鼠标状态、窗口尺寸和 SDL/OpenGL 资源。以全局变量 `V` 的形式作为单例使用。
- **关键成员**:
  - `initf` (`WindowInitFunc`) -- 初始化回调，在首次显示时调用
  - `exitf` (`WindowExitFunc`) -- 退出回调，通过 `atexit()` 注册以及在退出事件时调用
  - `resize` (`WindowResizeFunc`) -- 窗口大小改变回调
  - `display` (`WindowDisplayFunc`) -- 显示/绘制回调，每帧调用
  - `keyboard` (`WindowKeyboardFunc`) -- 键盘输入回调
  - `motion` (`WindowMotionFunc`) -- 鼠标移动回调，传递增量位移和按键状态
  - `first_display` (`bool`) -- 标记是否为首次显示，用于触发初始化
  - `redraw` (`bool`) -- 标记是否需要重绘
  - `mouseX`, `mouseY` (`int`) -- 当前鼠标位置
  - `mouseBut0`, `mouseBut2` (`int`) -- 左键（0）和右键（2）的按下状态
  - `width`, `height` (`int`) -- 窗口尺寸
  - `window` (`SDL_Window*`) -- SDL 窗口句柄
  - `gl_context` (`SDL_GLContext`) -- OpenGL 上下文
  - `gl_context_mutex` (`thread_mutex`) -- GL 上下文访问互斥锁

## 枚举与常量

### 函数指针类型定义（头文件）

- `WindowInitFunc` = `void (*)()` -- 初始化回调类型
- `WindowExitFunc` = `void (*)()` -- 退出回调类型
- `WindowResizeFunc` = `void (*)(int, int)` -- 窗口尺寸改变回调类型，参数为新的宽高
- `WindowDisplayFunc` = `void (*)()` -- 显示回调类型
- `WindowKeyboardFunc` = `void (*)(unsigned char)` -- 键盘回调类型，参数为按键字符
- `WindowMotionFunc` = `void (*)(int, int, int)` -- 鼠标移动回调类型，参数为 (deltaX, deltaY, button)

## 核心函数

### 公共接口（头文件声明）

#### `window_main_loop(title, width, height, initf, exitf, resize, display, keyboard, motion)`

- **功能**: 窗口主循环入口，创建 SDL 窗口和 OpenGL 上下文，进入事件循环直到退出
- **参数**: 窗口标题、初始宽高、以及六个回调函数
- **流程**:
  1. 初始化全局状态 `V`，保存所有回调
  2. 调用 `SDL_Init(SDL_INIT_VIDEO)` 初始化 SDL
  3. 配置 OpenGL Core Profile，启用上下文共享
  4. 创建可调整大小的 SDL 窗口（`SDL_WINDOW_RESIZABLE | SDL_WINDOW_OPENGL`）
  5. 创建 OpenGL 上下文，随后立即释放当前绑定（为多线程准备）
  6. 执行初始的 `window_reshape` 和 `window_display`
  7. 进入事件循环，处理文本输入、鼠标移动/点击、窗口大小改变、退出等事件
  8. 循环中通过 `SDL_WaitEventTimeout(nullptr, 100)` 以 100ms 超时等待事件，平衡响应性与 CPU 占用
  9. 退出时清理 GL 上下文、销毁窗口、调用 `SDL_Quit()`

#### `window_display_info(info)`

- **功能**: 在窗口中显示信息文本（当前通过日志输出替代图形渲染文本）

#### `window_display_help()`

- **功能**: 显示帮助信息，列出所有快捷键（h/r/p/esc/q/i/WASD/0-3 等）和鼠标操作说明

#### `window_redraw()`

- **功能**: 请求窗口重绘，设置 `V.redraw = true`，在下一次事件循环迭代时触发重绘

#### `window_opengl_context_enable()`

- **功能**: 获取 GL 上下文互斥锁并激活 OpenGL 上下文（`SDL_GL_MakeCurrent`）
- **返回**: 始终返回 `true`
- **线程安全**: 通过 `gl_context_mutex` 确保同一时刻只有一个线程持有 GL 上下文

#### `window_opengl_context_disable()`

- **功能**: 释放 OpenGL 上下文绑定并解锁互斥锁

### 内部函数（文件作用域静态函数）

#### `window_display_text(x, y, text)`

- **功能**: 文本渲染（图形渲染功能已禁用，当前仅在文本内容变化时通过日志输出）

#### `window_display()`

- **功能**: 帧显示函数。首次调用时触发 `initf` 和 `atexit(exitf)` 注册。激活 GL 上下文，设置正交投影矩阵，清除颜色/深度缓冲，调用 `display` 回调，交换缓冲区

#### `window_reshape(width, height)`

- **功能**: 处理窗口大小改变，在尺寸变化时调用 `resize` 回调并更新存储的宽高

#### `window_keyboard(key)`

- **功能**: 键盘事件处理，分发到 `keyboard` 回调。按 `q` 键时调用退出回调并返回 `true` 通知主循环退出

#### `window_mouse(button, state, x, y)`

- **功能**: 鼠标按键事件处理，跟踪左键和右键的按下/释放状态及当前位置

#### `window_motion(x, y)`

- **功能**: 鼠标移动事件处理，计算相对于上次位置的增量位移，调用 `motion` 回调传递增量和按键标识（0=左键, 2=右键）

## 依赖关系

- **内部头文件**:
  - `app/opengl/window.h` -- 自身头文件
  - `util/log.h` -- 日志工具（`LOG_ERROR`、`LOG_INFO_IMPORTANT`）
  - `util/string.h` -- 字符串工具
  - `util/thread.h` -- 线程工具（`thread_mutex`）
  - `util/version.h` -- 版本信息（`CYCLES_VERSION_STRING`）
- **外部库**:
  - `<SDL.h>` -- SDL 2 库，提供窗口创建、事件处理、OpenGL 上下文管理
  - `<epoxy/gl.h>` -- OpenGL 函数加载库
  - `<cstdio>`, `<cstdlib>` -- C 标准库
- **被引用**:
  - `src/app/cycles_standalone.cpp` -- 独立应用入口调用 `window_main_loop` 等函数

## 实现细节 / 关键算法

### 全局单例窗口状态

全局变量 `V`（`Window` 类型）以单例形式管理窗口的所有状态。这种设计简化了 C 风格回调函数的实现（SDL 回调无法携带上下文指针），但限制了同时只能存在一个窗口实例。

### OpenGL 上下文线程安全

`window_opengl_context_enable()` 和 `window_opengl_context_disable()` 通过 `thread_mutex` 实现互斥访问。这对于 Cycles 的多线程架构至关重要：渲染线程需要在 `OpenGLDisplayDriver::update_begin()` 中激活 GL 上下文写入 PBO，而主线程需要在绘制时激活同一上下文。互斥锁确保了这两个操作不会并发执行。

创建 GL 上下文后立即调用 `SDL_GL_MakeCurrent(V.window, nullptr)` 释放绑定，使其可以从任何线程重新激活。

### 延迟初始化

`initf` 回调在首次 `window_display()` 调用时执行（而非在 `window_main_loop` 开头），确保 OpenGL 上下文和窗口已完全就绪。`exitf` 通过 `atexit()` 注册，保证即使异常退出也能执行清理。

### 事件循环策略

主循环使用 `SDL_WaitEventTimeout(nullptr, 100)` 而非忙等待，在无事件时每 100ms 唤醒一次检查重绘标志。这在保持响应性的同时降低了空闲时的 CPU 占用。重绘通过 `V.redraw` 标志按需触发。

### 鼠标交互模型

鼠标移动回调接收增量位移（deltaX, deltaY）而非绝对坐标，简化了摄像机平移/旋转的实现。通过第三个参数（0=左键, 2=右键）区分移动和旋转操作。

### 文本渲染降级

原始设计使用 OpenGL 位图字体渲染文本（代码被 `#if 0` 禁用），当前降级为日志输出。帮助信息和渲染状态通过 `LOG_INFO_IMPORTANT` 输出到控制台。

## 关联文件

- `src/app/opengl/display_driver.h` / `display_driver.cpp` -- `OpenGLDisplayDriver` 使用 `window_opengl_context_enable/disable` 作为 GL 上下文回调
- `src/app/opengl/shader.h` / `shader.cpp` -- 显示着色器，在 `window_display()` 触发的绘制链中被调用
- `src/app/cycles_standalone.cpp` -- 独立应用入口，调用 `window_main_loop` 启动窗口并传入渲染回调
- `src/util/thread.h` -- 提供 `thread_mutex` 用于 GL 上下文同步
