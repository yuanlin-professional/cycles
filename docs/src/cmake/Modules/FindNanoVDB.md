# FindNanoVDB.cmake - NanoVDB 查找模块

## 概述

该模块用于查找 NanoVDB 库。NanoVDB 是 OpenVDB 的轻量级 GPU 友好版本，提供了一种紧凑的只读 VDB 数据结构表示，可以高效地在 GPU 上访问稀疏体积数据。在 Cycles 渲染器中，NanoVDB 用于 GPU 上的体积渲染，如烟雾、火焰和云等效果。

## 查找逻辑

模块按照以下方式搜索 NanoVDB：

1. **用户指定路径**：若定义了 `NANOVDB_ROOT_DIR`（CMake 变量或环境变量），则在该路径下搜索。

### 头文件搜索

在搜索目录的 `include` 子目录下查找 `nanovdb/NanoVDB.h`。

### 库文件搜索

NanoVDB 是纯头文件库，因此该模块**不搜索任何库文件**。

### 版本要求

该模块未设置特定的版本要求。

## 输出变量

| 变量名 | 说明 |
|--------|------|
| `NANOVDB_FOUND` | 布尔值，若为 `FALSE` 则表示未找到 NanoVDB |
| `NANOVDB_INCLUDE_DIRS` | NanoVDB 头文件目录 |
| `NANOVDB_ROOT_DIR` | 搜索 NanoVDB 的根目录，可通过环境变量设置 |

### 内部变量（高级）

| 变量名 | 说明 |
|--------|------|
| `NANOVDB_INCLUDE_DIR` | NanoVDB 头文件目录（单数形式） |

## 依赖关系

该模块使用 CMake 内置的 `FindPackageHandleStandardArgs` 模块来处理标准的查找参数。无其他外部 Find 模块依赖。

> **注意**：NanoVDB 是纯头文件库（header-only），使用时仅需包含头文件目录，无需链接额外的库文件。
