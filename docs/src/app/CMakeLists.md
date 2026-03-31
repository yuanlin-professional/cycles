# CMakeLists.txt - Cycles 应用层 CMake 构建配置

## 概述

`src/app/CMakeLists.txt` 是 Cycles 应用层（app 模块）的 CMake 构建脚本，负责定义和配置两个可执行目标：`cycles`（独立渲染器）和 `cycles_precompute`（预计算查找表生成工具）。该文件管理源文件列表、头文件搜索路径、库链接依赖以及平台特定的构建选项，是整个独立应用构建流程的入口配置。

## 核心构建目标

### cycles（独立渲染器可执行文件）
- **启用条件**: `WITH_CYCLES_STANDALONE`
- **源文件**:
  - `cycles_standalone.cpp` — 主程序入口
  - `cycles_xml.cpp` / `cycles_xml.h` — XML 场景解析
  - `oiio_output_driver.cpp` / `oiio_output_driver.h` — OIIO 图像输出驱动
  - GUI 模式额外源文件（`WITH_CYCLES_STANDALONE_GUI`）：`opengl/display_driver.cpp/.h`、`opengl/shader.cpp/.h`、`opengl/window.cpp/.h`

### cycles_precompute（预计算工具可执行文件）
- **启用条件**: `WITH_CYCLES_PRECOMPUTE`
- **源文件**: `cycles_precompute.cpp`

## 库依赖配置

### 核心链接库（LIB）
始终链接的 Cycles 内部模块库：
- `cycles_device` — 渲染设备（Device）抽象层
- `cycles_kernel` — 渲染内核（Kernel）
- `cycles_scene` — 场景（Scene）管理
- `cycles_session` — 渲染会话管理
- `cycles_bvh` — 层次包围体（BVH）加速结构
- `cycles_subd` — 细分曲面
- `cycles_graph` — 节点图系统
- `cycles_util` — 通用工具库

### 条件依赖
- **OSL 支持** (`WITH_CYCLES_OSL`): 追加 `cycles_kernel_osl`
- **天空模型**: 独立仓库模式下链接 `extern_sky`，Blender 集成模式下链接 `bf_intern_sky`
- **GUI 支持** (`WITH_CYCLES_STANDALONE_GUI`): 追加 Epoxy 和 SDL2 的头文件路径和库
- **USD 支持** (`WITH_USD`): 追加 USD 头文件路径、`cycles_hydra` 库和 USD 库，同时添加 TBB 废弃头文件警告抑制宏
- `cycles_external_libraries_append(LIB)` — 追加所有外部库依赖

## 平台特定配置

### macOS (APPLE)
- 设置 `BUILD_WITH_INSTALL_RPATH` 属性为 true
- GUI 模式下追加 SDL 所需的系统框架链接：AudioToolbox、AudioUnit、Cocoa、CoreAudio、CoreHaptics、CoreVideo、ForceFeedback、GameController

## 安装与测试

### 安装规则
- **cycles 可执行文件**: 安装到 `CMAKE_INSTALL_PREFIX`
- **cycles_precompute 可执行文件**: 安装到 `CMAKE_INSTALL_PREFIX`
- **USD 资源**（仅独立仓库模式 + USD 支持）: 安装 USD 运行时插件目录

### CTest 测试
- `cycles_version` — 运行 `cycles --version` 验证可执行文件可正常启动

## 关键 CMake 变量

| 变量名 | 说明 |
|-------|------|
| `WITH_CYCLES_STANDALONE` | 启用独立渲染器构建 |
| `WITH_CYCLES_STANDALONE_GUI` | 启用 GUI 交互模式（需要 Epoxy + SDL2） |
| `WITH_CYCLES_PRECOMPUTE` | 启用预计算工具构建 |
| `WITH_CYCLES_OSL` | 启用 OSL 着色器系统支持 |
| `WITH_USD` | 启用 USD 场景格式支持 |
| `CYCLES_STANDALONE_REPOSITORY` | 标识是否为独立仓库构建（非 Blender 集成） |

## 依赖关系

### 被引用
- 上级 `CMakeLists.txt` 通过 `add_subdirectory(app)` 引入本文件

### 引用的 CMake 模块/宏
- `cycles_external_libraries_append()` — Cycles 自定义宏，追加外部库依赖
- `cycles_install_libraries()` — Cycles 自定义宏，安装运行时所需的共享库

## 关联文件

- `src/app/cycles_standalone.cpp` — cycles 可执行文件的主入口
- `src/app/cycles_precompute.cpp` — cycles_precompute 可执行文件的主入口
- `src/app/opengl/` — GUI 模式的 OpenGL 相关源文件
