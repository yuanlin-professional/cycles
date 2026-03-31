# FindLibEpoxy.cmake - LibEpoxy 查找模块

## 概述

该模块用于查找 libepoxy 库。libepoxy 与 Epoxy 是同一个库的不同命名引用，用于处理 OpenGL 函数指针的自动加载和管理。在 Cycles 渲染器中，libepoxy 用于 OpenGL 视口渲染中函数指针的跨平台管理。

## 查找逻辑

模块按照以下方式搜索 libepoxy：

1. **用户指定路径**：若定义了 `EPOXY_ROOT_DIR`（CMake 变量或环境变量），则在该路径下搜索。

### 头文件搜索

在搜索目录的 `include` 子目录下查找 `epoxy/gl.h`。

### 库文件搜索

在搜索目录的 `lib64`、`lib` 子目录下查找名为 `epoxy` 的库文件。

### 版本要求

该模块未设置特定的版本要求。

## 输出变量

| 变量名 | 说明 |
|--------|------|
| `LibEpoxy_FOUND` | 布尔值，若为 `FALSE` 则表示未找到 libepoxy |
| `LibEpoxy_INCLUDE_DIRS` | libepoxy 头文件目录 |
| `LibEpoxy_LIBRARIES` | 链接 libepoxy 所需的库文件列表 |
| `LibEpoxy_ROOT_DIR` | 搜索 libepoxy 的根目录，可通过环境变量设置 |

### 内部变量（高级）

| 变量名 | 说明 |
|--------|------|
| `LibEpoxy_INCLUDE_DIR` | libepoxy 头文件目录（单数形式） |
| `LibEpoxy_LIBRARY` | libepoxy 库文件路径（单数形式） |

## 依赖关系

该模块使用 CMake 内置的 `FindPackageHandleStandardArgs` 模块来处理标准的查找参数。无其他外部 Find 模块依赖。

> **注意**：该模块与 `FindEpoxy.cmake` 功能相同，但使用不同的变量命名约定（`LibEpoxy_*` vs `Epoxy_*`）。两个模块共享相同的根目录环境变量 `EPOXY_ROOT_DIR`。选择使用哪个模块取决于项目中 `find_package()` 的调用方式。
