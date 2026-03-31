# CMakeLists.txt - 文档目录顶层构建配置

## 概述

此文件是 Cycles 渲染器 `src/doc/` 目录下的顶层 CMake 构建脚本。它作为文档子目录的入口点，负责将构建流程递归地分发到各个子目录中。

该文件采用 Apache-2.0 许可证，版权归属 Blender Foundation（2011-2022）。

## 构建选项 / 变量

此文件未定义任何构建变量或选项，仅包含一个子目录引用。

### 子目录引用

| 子目录 | 说明 |
|--------|------|
| `license` | 包含 Cycles 项目使用的各类开源许可证文件及其安装规则 |

## 依赖关系

- **上游依赖**: 此文件由 Cycles 项目的上级 `CMakeLists.txt` 调用
- **下游依赖**: 通过 `add_subdirectory(license)` 调用 `license/CMakeLists.txt`，该子目录负责许可证文件的安装部署
