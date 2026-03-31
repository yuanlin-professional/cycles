# FindGLEW.cmake - GLEW 查找模块

## 概述

该模块用于查找 GLEW（OpenGL Extension Wrangler Library）库。GLEW 是一个跨平台的 OpenGL 扩展加载库，用于查询和加载 OpenGL 扩展。在 Cycles 渲染器中，GLEW 用于管理 OpenGL 扩展函数的加载，支持 GPU 视口渲染功能。

## 查找逻辑

模块按照以下方式搜索 GLEW：

1. **用户指定路径**：若定义了 `GLEW_ROOT_DIR`（CMake 变量或环境变量），则在该路径下搜索。

### 头文件搜索

在搜索目录的 `include` 子目录下查找 `GL/glew.h`。

### 库文件搜索

在搜索目录的 `lib64`、`lib` 子目录下查找名为 `GLEW` 的库文件。

### 版本要求

该模块未设置特定的版本要求。

## 输出变量

| 变量名 | 说明 |
|--------|------|
| `GLEW_FOUND` | 布尔值，若为 `FALSE` 则表示未找到 GLEW |
| `GLEW_INCLUDE_DIRS` | GLEW 头文件目录，在 `GLEW_INCLUDE_DIR` 被找到时设置 |
| `GLEW_ROOT_DIR` | 搜索 GLEW 的根目录，可通过环境变量设置 |

### 内部变量（高级）

| 变量名 | 说明 |
|--------|------|
| `GLEW_INCLUDE_DIR` | GLEW 头文件目录（单数形式） |
| `GLEW_LIBRARY` | GLEW 库文件路径 |

> **注意**：该模块找到 GLEW 后仅设置 `GLEW_INCLUDE_DIRS`，不设置 `GLEW_LIBRARIES` 复数变量。使用时需直接引用 `GLEW_LIBRARY`。

## 依赖关系

该模块使用 CMake 内置的 `FindPackageHandleStandardArgs` 模块来处理标准的查找参数。无其他外部 Find 模块依赖。
