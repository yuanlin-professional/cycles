# CMakeLists.txt - 许可证文件安装构建配置

## 概述

此文件是 Cycles 渲染器许可证目录（`src/doc/license/`）的 CMake 构建脚本。它的主要功能是定义需要安装的许可证文件列表，并通过 `delayed_install` 命令将这些文件部署到 Cycles 的安装路径下。

该文件采用 Apache-2.0 许可证，版权归属 Blender Foundation（2011-2022）。

## 构建选项 / 变量

### LICENSES 变量

定义了需要安装的许可证文件列表：

| 文件名 | 说明 |
|--------|------|
| `Apache2-license.txt` | Apache 2.0 许可证（Cycles 默认许可证） |
| `BSD-3-Clause-license.txt` | BSD 三条款许可证 |
| `MIT-license.txt` | MIT 许可证 |
| `readme.txt` | 许可证说明文件 |
| `SPDX-license-identifiers.txt` | SPDX 许可证标识符对照表 |
| `Zlib-license.txt` | Zlib 许可证 |

### 安装规则

使用 `delayed_install` 函数将上述许可证文件安装到 `${CYCLES_INSTALL_PATH}/license` 目录下。`delayed_install` 是 Cycles/Blender 构建系统提供的自定义 CMake 函数，用于延迟执行文件安装操作，以便在构建完成后统一处理文件部署。

## 依赖关系

- **上游依赖**: 由 `src/doc/CMakeLists.txt` 通过 `add_subdirectory(license)` 调用
- **外部变量**: 依赖 `CYCLES_INSTALL_PATH` 变量（由上层构建系统定义，指定 Cycles 的安装路径）
- **构建函数**: 依赖 `delayed_install` 函数（由 Cycles/Blender 构建系统提供）
