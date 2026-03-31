# cycles_standalone.cpp - Cycles 独立渲染器主程序入口

## 概述

`cycles_standalone.cpp` 是 Cycles 渲染引擎的独立可执行程序入口文件。它提供了一个脱离 Blender 的命令行渲染工具，支持从 XML 或 USD 场景描述文件加载场景并执行路径追踪渲染。该文件同时包含可选的 GUI 交互模式（基于 OpenGL/SDL），允许用户在窗口中实时预览渲染结果并通过键盘和鼠标进行相机操控。

## 类与结构体

### Options
- **功能**: 全局配置结构体，存储渲染会话的所有运行参数和状态
- **关键成员**:
  - `session` (`unique_ptr<Session>`) — 渲染会话对象，管理整个渲染流程的生命周期
  - `scene` (`Scene*`) — 场景（Scene）指针，指向当前渲染场景
  - `filepath` (`string`) — 输入场景文件路径（XML 或 USD 格式）
  - `width, height` (`int`) — 输出图像的宽度和高度（像素），默认 1024x512
  - `scene_params` (`SceneParams`) — 场景参数，包括着色器系统选择（SVM/OSL）
  - `session_params` (`SessionParams`) — 会话参数，包括设备（Device）选择、采样数、线程数等
  - `quiet` (`bool`) — 静默模式，后台渲染时不输出进度信息
  - `show_help, interactive, pause` (`bool`) — GUI 模式下的交互状态标志
  - `output_filepath` (`string`) — 输出图像文件路径
  - `output_pass` (`string`) — 输出的渲染通道名称，默认为 `"combined"`

## 核心函数

### session_print()
- **签名**: `static void session_print(const string &str)`
- **功能**: 向标准输出打印状态信息，使用回车符 `\r` 覆盖上一行输出，并自动补空格以清除残留字符

### session_print_status()
- **签名**: `static void session_print_status()`
- **功能**: 获取当前渲染进度百分比和状态文本，格式化后调用 `session_print()` 输出

### session_buffer_params()
- **签名**: `static BufferParams &session_buffer_params()`
- **功能**: 根据当前 `options.width` 和 `options.height` 构建并返回 `BufferParams`，定义渲染缓冲区的尺寸

### scene_init()
- **签名**: `static void scene_init()`
- **功能**: 初始化场景。根据文件扩展名选择 XML 解析器（`xml_read_file`）或 USD 读取器（`HdCyclesFileReader::read`）加载场景数据。若命令行指定了宽高则覆盖相机设置，否则从场景相机读取。最后计算自动视平面（auto viewplane）

### session_init()
- **签名**: `static void session_init()`
- **功能**: 创建渲染会话（Session）并配置输出驱动。在 GUI 模式下设置 `OpenGLDisplayDriver` 用于窗口显示；若指定了输出文件路径则设置 `OIIOOutputDriver` 用于写入图像文件。设置进度回调后调用 `scene_init()` 加载场景，添加 `PASS_COMBINED` 渲染通道，最后调用 `session->reset()` 和 `session->start()` 启动渲染

### session_exit()
- **签名**: `static void session_exit()`
- **功能**: 释放渲染会话资源，打印"Finished Rendering."结束信息

### display_info() (仅 GUI 模式)
- **签名**: `static void display_info(Progress &progress)`
- **功能**: 在 GUI 窗口标题栏显示渲染状态信息，包括总时间、帧延迟、进度百分比、单次采样平均耗时和交互模式开关状态

### display() (仅 GUI 模式)
- **签名**: `static void display()`
- **功能**: GUI 模式下的显示回调，调用 `session->draw()` 绘制渲染结果并更新状态信息

### motion() (仅 GUI 模式)
- **签名**: `static void motion(const int x, const int y, int button)`
- **功能**: 鼠标拖动回调。左键（button=0）平移相机，右键（button=2）旋转相机。修改相机矩阵后重置渲染会话

### resize() (仅 GUI 模式)
- **签名**: `static void resize(const int width, const int height)`
- **功能**: 窗口尺寸变化回调，同步更新相机分辨率和视平面并重置渲染

### keyboard() (仅 GUI 模式)
- **签名**: `static void keyboard(unsigned char key)`
- **功能**: 键盘输入回调，支持以下快捷键：
  - `h` — 切换帮助信息显示
  - `r` — 重置渲染
  - `Esc` — 取消渲染
  - `p` — 暂停/继续渲染
  - `i` — 切换交互模式
  - `w/a/s/d` — 交互模式下的相机前后左右平移
  - `0/1/2/3` — 交互模式下设置最大光线反弹次数

### options_parse()
- **签名**: `static void options_parse(const int argc, const char **argv)`
- **功能**: 解析命令行参数。使用 OIIO 的 `ArgParse` 库处理以下选项：
  - `--device` — 渲染设备（CPU、CUDA、HIP、OptiX、oneAPI、Metal 等）
  - `--shadingsys` — 着色器系统（svm 或 osl，仅在编译了 OSL 时可用）
  - `--background` — 后台渲染模式（无 GUI）
  - `--quiet` — 静默模式
  - `--samples` — 采样数
  - `--output` — 输出文件路径
  - `--threads` — CPU 渲染线程数
  - `--width / --height` — 图像分辨率
  - `--tile-size` — 分块大小
  - `--list-devices` — 列出可用渲染设备
  - `--profile` — 启用性能分析
  - `--log-level` — 日志级别（fatal/error/warning/info/stats/debug）
  - `--version` — 打印版本号

  函数还包含参数验证逻辑：检查设备（Device）可用性、OSL 只能在 CPU 上使用、采样数不能为负等

### main()
- **签名**: `int main(const int argc, const char **argv)`
- **功能**: 程序入口。依次执行日志初始化（`log_init`）、路径初始化（`path_init`）、命令行解析。后台模式下直接启动渲染会话并等待完成；GUI 模式下通过 `window_main_loop` 启动带有 OpenGL 窗口的事件循环

## 依赖关系

### 内部头文件
- `device/device.h` — 渲染设备抽象层
- `scene/camera.h` — 相机
- `scene/integrator.h` — 积分器
- `scene/scene.h` — 场景管理
- `session/buffers.h` — 渲染缓冲区
- `session/session.h` — 渲染会话
- `util/args.h` — 命令行参数解析（封装 OIIO ArgParse）
- `util/log.h` — 日志系统
- `util/path.h` — 文件路径工具
- `util/progress.h` — 渲染进度追踪
- `util/string.h` — 字符串工具
- `util/unique_ptr.h` — 智能指针
- `util/version.h` — 版本信息
- `app/cycles_xml.h` — XML 场景文件读取
- `app/oiio_output_driver.h` — OIIO 图像输出驱动

### 条件编译依赖
- `util/time.h`, `util/transform.h` — 仅 GUI 模式（`WITH_CYCLES_STANDALONE_GUI`）
- `hydra/file_reader.h` — USD 支持（`WITH_USD`）
- `opengl/display_driver.h`, `opengl/window.h` — OpenGL GUI 显示（`WITH_CYCLES_STANDALONE_GUI`）

### 外部库
- OIIO（OpenImageIO）— 通过 `ArgParse` 用于命令行解析

## 实现细节 / 关键算法

1. **条件编译架构**: 文件大量使用 `WITH_CYCLES_STANDALONE_GUI`、`WITH_USD`、`WITH_OSL` 等宏进行条件编译，使同一份代码同时支持纯命令行后台渲染和带 GUI 的交互渲染两种模式。

2. **渲染流程**:
   - 解析命令行 -> 创建 Session -> 设置输出驱动 -> 加载场景 -> 添加渲染通道 -> 启动渲染 -> 等待完成 -> 输出图像 -> 释放资源

3. **GUI 交互模式**: 交互模式下支持通过鼠标拖拽旋转/平移相机、WASD 键盘导航、以及动态调整最大光线反弹次数（0-3 次），每次交互后均重置渲染会话以重新采样。

4. **设备选择策略**: `options_parse` 中通过 `Device::available_types()` 查询编译时支持的设备类型列表，并在用户指定的设备类型中选取第一个可用设备。

## 关联文件

- `src/app/cycles_xml.h` / `src/app/cycles_xml.cpp` — XML 场景加载
- `src/app/oiio_output_driver.h` / `src/app/oiio_output_driver.cpp` — 图像输出
- `src/app/opengl/display_driver.h` — GUI 显示驱动
- `src/app/opengl/window.h` — GUI 窗口管理
- `src/session/session.h` — 渲染会话核心
- `src/hydra/file_reader.h` — USD 文件读取（可选）
