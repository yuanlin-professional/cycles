# FindEpoxy.cmake - Epoxy 查找模块

## 概述

该模块用于查找 Epoxy 库。Epoxy 是一个用于处理 OpenGL 函数指针管理的库，它自动解析 OpenGL、OpenGL ES 和 EGL 的函数指针加载问题。在 Cycles 渲染器中，Epoxy 用于 OpenGL 视口渲染的函数指针管理。

## 查找逻辑

模块按照以下方式搜索 Epoxy：

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
| `Epoxy_FOUND` | 布尔值，若为 `FALSE` 则表示未找到 Epoxy |
| `Epoxy_INCLUDE_DIRS` | Epoxy 头文件目录 |
| `Epoxy_LIBRARIES` | 链接 Epoxy 所需的库文件列表 |
| `Epoxy_ROOT_DIR` | 搜索 Epoxy 的根目录，可通过环境变量设置 |

### 内部变量（高级）

| 变量名 | 说明 |
|--------|------|
| `Epoxy_INCLUDE_DIR` | Epoxy 头文件目录（单数形式） |
| `Epoxy_LIBRARY` | Epoxy 库文件路径（单数形式） |

## 依赖关系

该模块使用 CMake 内置的 `FindPackageHandleStandardArgs` 模块来处理标准的查找参数。无其他外部 Find 模块依赖。

> **注意**：该模块使用混合大小写变量名（如 `Epoxy_FOUND` 而非 `EPOXY_FOUND`），但根目录变量使用大写 `EPOXY_ROOT_DIR`。另有一个类似的 `FindLibEpoxy.cmake` 模块，变量命名为 `LibEpoxy_*` 形式。
