# FindGflags.cmake - Google gflags 查找模块

## 概述

该模块用于查找 Google gflags 命令行参数解析库。gflags 提供了一种便捷的方式来定义和解析命令行标志参数。该模块最初来源于 Ceres Solver 项目。在 Cycles 渲染器中，gflags 可用于命令行工具的参数处理。

## 查找逻辑

该模块采用两阶段查找策略：

### 第一阶段：导出的 CMake 配置查找

若 `GFLAGS_PREFER_EXPORTED_GFLAGS_CMAKE_CONFIGURATION` 为 `TRUE`（默认行为，除非用户定义了 `GFLAGS_INCLUDE_DIR_HINTS` 或 `GFLAGS_LIBRARY_DIR_HINTS`），模块优先查找 gflags >= 2.1 版本导出的 CMake 配置文件：

1. 首先使用 `NO_CMAKE_PACKAGE_REGISTRY` 搜索已安装的版本
2. 若失败，再搜索已导出的构建目录

模块还会处理 gflags v2.1 - v2.1.2 中已知的 CMake 配置缺陷（导入目标名称问题）。

### 第二阶段：手动组件搜索

若第一阶段未找到 gflags，模块执行手动搜索，搜索路径包括：

1. `GFLAGS_ROOT_DIR` 指定的路径
2. `/usr/local/include`、`/usr/local/homebrew/include`（macOS）
3. `/opt/local/var/macports/software`（macOS MacPorts）
4. `/opt/local/include`、`/usr/include`
5. `/opt/lib/gflags/include`
6. Windows 下 `C:/Program Files` 前缀的路径

### 命名空间检测

模块会自动检测 gflags 的命名空间（`google` 或 `gflags`），采用两种方法：

1. **编译测试法**：使用 `check_cxx_source_compiles()` 编译测试程序
2. **正则匹配法**：解析 `gflags.h` 和 `gflags_declare.h` 头文件

### 版本要求

无强制版本要求，但能自动检测版本并处理不同版本间的兼容性问题。

## 输出变量

| 变量名 | 说明 |
|--------|------|
| `GFLAGS_FOUND` | 布尔值，若为 `TRUE` 则表示找到 gflags |
| `GFLAGS_INCLUDE_DIRS` | gflags 头文件目录 |
| `GFLAGS_LIBRARIES` | 链接 gflags 所需的库文件列表（包括线程库等依赖） |
| `GFLAGS_NAMESPACE` | gflags 的命名空间（`google` 或 `gflags`） |
| `GFLAGS_ROOT_DIR` | 搜索 gflags 的根目录，可通过环境变量设置 |

### 控制变量

| 变量名 | 说明 |
|--------|------|
| `GFLAGS_PREFER_EXPORTED_GFLAGS_CMAKE_CONFIGURATION` | 是否优先使用导出的 CMake 配置 |
| `GFLAGS_INCLUDE_DIR_HINTS` | 额外的头文件搜索目录列表 |
| `GFLAGS_LIBRARY_DIR_HINTS` | 额外的库文件搜索目录列表 |

### 内部变量（高级）

| 变量名 | 说明 |
|--------|------|
| `GFLAGS_INCLUDE_DIR` | gflags 头文件目录（单数形式） |
| `GFLAGS_LIBRARY` | gflags 库文件路径（单数形式） |

## 依赖关系

- **Threads**：gflags 通常需要线程库支持，模块会通过 `find_package(Threads)` 查找
- **shlwapi**：在 Windows 平台上，若可用会链接 `shlwapi.lib`
- 使用 CMake 内置的 `FindPackageHandleStandardArgs`、`CheckCXXSourceCompiles`、`CheckIncludeFileCXX` 模块
