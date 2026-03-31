# app - 独立应用程序

## 概述

`src/app/` 模块是 Cycles 渲染引擎的独立（standalone）应用程序入口，提供命令行渲染器功能。该模块允许用户在不依赖 Blender 宿主程序的情况下，直接通过命令行调用 Cycles 渲染 XML 或 USD 场景文件，并将结果输出为图像文件。同时，在编译启用 GUI 支持时，可通过 OpenGL 窗口进行交互式视口预览。

## 目录结构

| 文件名 | 类型 | 说明 |
|--------|------|------|
| `CMakeLists.txt` | 构建配置 | CMake 构建脚本，定义 `cycles` 和 `cycles_precompute` 两个可执行目标，管理依赖库链接（cycles_device、cycles_kernel、cycles_scene、cycles_session、cycles_bvh 等） |
| `cycles_standalone.cpp` | C++ 源文件 | 主入口文件，包含 `main()` 函数、命令行参数解析（`options_parse`）、会话初始化/退出（`session_init`/`session_exit`）、场景加载（`scene_init`）以及 GUI 模式下的交互回调（显示、键盘、鼠标、窗口缩放） |
| `cycles_xml.cpp` | C++ 源文件 | XML 场景文件读取器，解析 Cycles 自定义 XML 格式，支持读取相机、着色器、着色器图、背景、网格（含细分曲面）、灯光、变换、状态及嵌套 include |
| `cycles_xml.h` | C++ 头文件 | XML 读取接口声明，导出 `xml_read_file()` 函数及角度转换宏 `RAD2DEGF`/`DEG2RADF` |
| `cycles_precompute.cpp` | C++ 源文件 | 预计算工具，通过蒙特卡洛采样计算 GGX 微表面 BRDF/BSDF 的反照率查找表（LUT），包括 `ggx_E`、`ggx_Eavg`、`ggx_glass_E`、`ggx_gen_schlick_s` 等项，输出为 C++ 静态数组格式 |
| `oiio_output_driver.cpp` | C++ 源文件 | 基于 OpenImageIO 的输出驱动实现，负责将渲染结果写入图像文件，支持自动 gamma 校正（sRGB 色彩空间检测）及图像垂直翻转 |
| `oiio_output_driver.h` | C++ 头文件 | `OIIOOutputDriver` 类声明，继承自 `OutputDriver`，提供 `write_render_tile()` 接口 |
| `io_export_cycles_xml.py` | Python 脚本 | Blender 插件脚本，用于将 Blender 中的活动物体网格导出为 Cycles XML 格式（顶点、面、UV），主要用于生成测试文件 |

## 核心功能

### 命令行选项

独立渲染器通过 `cycles_standalone.cpp` 中的 `options_parse()` 解析以下命令行参数：

- `--device <DEVICE>` - 指定渲染设备（CPU、CUDA、HIP、OPTIX、METAL 等）
- `--shadingsys <svm|osl>` - 选择着色系统（需编译 OSL 支持）
- `--background` - 后台渲染模式（无 GUI）
- `--quiet` - 静默模式，不输出进度信息
- `--samples <N>` - 设置渲染采样数
- `--output <PATH>` - 输出图像文件路径
- `--threads <N>` - CPU 渲染线程数
- `--width <N>` / `--height <N>` - 图像分辨率
- `--tile-size <N>` - 分块大小（像素）
- `--list-devices` - 列出可用渲染设备
- `--log-level <LEVEL>` - 日志级别（fatal、error、warning、info、stats、debug）
- `--profile` - 启用性能分析日志
- `--version` - 显示版本号

### 渲染管线

1. **参数解析** - `options_parse()` 处理命令行参数并配置 `SessionParams`/`SceneParams`
2. **会话创建** - `session_init()` 创建 `Session` 对象，设置输出驱动（`OIIOOutputDriver`）和显示驱动（GUI 模式下的 `OpenGLDisplayDriver`）
3. **场景加载** - `scene_init()` 通过 XML 或 USD 读取器加载场景数据
4. **渲染执行** - 调用 `session->start()` 启动渲染，后台模式下 `session->wait()` 等待完成
5. **结果输出** - `OIIOOutputDriver::write_render_tile()` 将渲染缓冲区写入磁盘

### 输出格式

通过 OpenImageIO 支持所有 OIIO 可写入的图像格式（PNG、EXR、TIFF、JPEG 等）。输出驱动会自动检测文件格式并对非线性色彩空间（如 sRGB）应用 gamma 校正（1/2.2）。

### 预计算工具

`cycles_precompute` 是独立的命令行工具，用于生成微表面 BSDF 能量守恒所需的查找表数据，通过并行蒙特卡洛积分计算。生成的数据直接输出为 C++ 源代码中的静态浮点数组。

## 依赖关系

### 上游依赖（本模块依赖）

- `cycles_device` - 设备抽象层（CPU/GPU）
- `cycles_kernel` - 渲染内核
- `cycles_scene` - 场景数据（相机、着色器、网格、灯光等）
- `cycles_session` - 渲染会话管理
- `cycles_bvh` - BVH 加速结构
- `cycles_subd` - 细分曲面
- `cycles_graph` - 节点图系统
- `cycles_util` - 工具库
- `cycles_hydra` - USD/Hydra 集成（可选，需 `WITH_USD`）
- `cycles_kernel_osl` - OSL 着色内核（可选，需 `WITH_CYCLES_OSL`）
- OpenImageIO - 图像读写
- SDL2 / Epoxy - GUI 窗口和 OpenGL（可选，需 `WITH_CYCLES_STANDALONE_GUI`）

### 下游依赖（依赖本模块）

- 无 - 本模块为最终可执行文件入口，不被其他 Cycles 模块依赖

## 参见

- `src/app/opengl/` - OpenGL 显示后端（交互式视口渲染）
- `src/session/` - 渲染会话和输出驱动基类
- `src/scene/` - 场景数据模型
- `src/graph/` - 节点 XML 序列化支持
