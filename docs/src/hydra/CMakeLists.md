# CMakeLists.txt - Hydra渲染代理模块构建配置

## 概述

本文件是 Cycles Hydra渲染代理 模块的 CMake 构建配置文件，定义了静态库 `cycles_hydra` 和可选的动态插件库 `hdCycles` 的编译规则、依赖链接和安装路径。静态库包含所有 Hydra 集成代码，动态库作为 USD 插件由 Hydra 框架动态加载。

## 类与结构体

无（构建配置文件）。

## 核心函数

无（构建配置文件）。

## 依赖关系

- **构建依赖**:
  - `cycles_scene` -- Cycles 场景管理库
  - `cycles_session` -- Cycles 会话管理库
  - `cycles_graph` -- Cycles 节点图库
  - `${USD_LIBRARIES}` -- Pixar USD 库
  - `${Epoxy_LIBRARIES}` -- OpenGL 加载库（用于显示驱动）
  - `cycles_external_libraries_append()` -- 追加外部依赖（OpenImageIO、OpenEXR 等）
- **包含路径**:
  - `..`（src 根目录）
  - `${USD_INCLUDE_DIRS}`
  - `${Epoxy_INCLUDE_DIRS}`
  - `${PYTHON_INCLUDE_DIRS}`

## 实现细节 / 关键算法

### 1. 静态库 cycles_hydra

源文件集合包含 18 个 `.cpp` 实现文件和 19 个头文件（含 `geometry.inl`）。当检测到 `${USD_INCLUDE_DIR}/pxr/imaging/hgiGL` 目录存在时，追加 `display_driver.cpp/.h` 并定义 `WITH_HYDRA_DISPLAY_DRIVER` 宏，启用 OpenGL 实时显示驱动支持。

编译选项：
- MSVC：禁用警告 C4003、C4244、C4506
- GCC：禁用 float-conversion、double-promotion、deprecated 警告
- 定义 `GLOG_NO_ABBREVIATED_SEVERITIES`、`OSL_DEBUG`、`TBB_USE_DEBUG`（调试模式下）
- MSVC 下定义 `NOMINMAX` 防止 Windows 头文件的 min/max 宏干扰
- 禁用 TBB 已弃用头文件的警告（`__TBB_show_deprecation_message_atomic_H` / `_task_H`）

### 2. 动态插件库 hdCycles（条件编译）

仅当 `WITH_CYCLES_HYDRA_RENDER_DELEGATE` 开启时构建。包含 `plugin.h` 和 `plugin.cpp`，链接 `cycles_hydra` 静态库。

平台特定处理：
- macOS：使用自定义导出符号表（`apple_symbols.map`），设置 `BUILD_WITH_INSTALL_RPATH`
- Linux：使用版本脚本（`linux_symbols.map`）控制符号可见性
- 所有平台：移除库名前缀（`PREFIX ""`）

### 3. 安装路径配置

根据构建目标选择安装位置：
- Houdini：`${CMAKE_INSTALL_PREFIX}/houdini/dso/usd_plugins`
- Blender 插件：`${CYCLES_INSTALL_PATH}/hydra`
- 独立安装：`${CMAKE_INSTALL_PREFIX}/hydra`

### 4. plugInfo.json 配置

使用 `configure_file()` 将模板中的 `@PLUG_INFO_ROOT@`、`@PLUG_INFO_LIBRARY_PATH@`、`@PLUG_INFO_RESOURCE_PATH@` 替换为实际路径，生成最终的 USD 插件元数据文件。Houdini 构建时还生成 `cycles.json` 包配置文件。

## 关联文件

- `resources/plugInfo.json` -- USD 插件元数据模板
- `resources/cycles.json` -- Houdini 包配置模板
- `resources/apple_symbols.map` -- macOS 导出符号表
- `resources/linux_symbols.map` -- Linux 版本脚本
- `plugin.h` / `plugin.cpp` -- 动态库入口源文件
