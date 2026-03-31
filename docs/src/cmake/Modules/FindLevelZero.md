# FindLevelZero.cmake - Level Zero 查找模块

## 概述

该模块用于查找 Intel Level Zero 库。Level Zero 是 Intel oneAPI 的底层硬件接口，提供对 Intel GPU 的直接访问能力。在 Cycles 渲染器中，Level Zero 用于支持 Intel GPU 的 oneAPI 渲染后端，实现在 Intel 独立显卡和集成显卡上的硬件加速渲染。

## 查找逻辑

模块按照以下优先级搜索 Level Zero：

1. **用户指定路径**：若定义了 `LEVEL_ZERO_ROOT_DIR`（CMake 变量或环境变量），则优先在该路径下搜索。
2. **系统默认路径**：
   - `/usr/lib`
   - `/usr/local/lib`

### 库文件搜索

在搜索目录的 `lib64`、`lib` 子目录下查找名为 `ze_loader` 的库文件。

### 头文件搜索

在搜索目录的 `include` 子目录下查找 `level_zero/ze_api.h`。

### 版本要求

该模块未设置特定的版本要求。

## 输出变量

| 变量名 | 说明 |
|--------|------|
| `LEVEL_ZERO_FOUND` | 布尔值，若为 `FALSE` 则表示未找到 Level Zero |
| `LEVEL_ZERO_LIBRARY` | Level Zero 加载器库文件路径 |
| `LEVEL_ZERO_INCLUDE_DIR` | Level Zero 头文件目录 |
| `LEVEL_ZERO_ROOT_DIR` | 搜索 Level Zero 的根目录，可通过环境变量设置 |

> **注意**：若未找到 Level Zero，`LEVEL_ZERO_LIBRARY` 和 `LEVEL_ZERO_INCLUDE_DIR` 会被取消设置（`unset`）。

### 内部变量（高级）

| 变量名 | 说明 |
|--------|------|
| `LEVEL_ZERO_LIBRARY` | Level Zero 库文件路径 |
| `LEVEL_ZERO_INCLUDE_DIR` | Level Zero 头文件目录 |

## 依赖关系

该模块使用 CMake 内置的 `FindPackageHandleStandardArgs` 模块来处理标准的查找参数。无其他外部 Find 模块依赖。
