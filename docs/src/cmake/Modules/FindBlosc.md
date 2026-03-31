# FindBlosc.cmake - Blosc 查找模块

## 概述

该模块用于查找 Blosc 压缩库。Blosc 是一个高性能的分块压缩库，专为二进制数据的快速压缩/解压缩而设计。在 Cycles 渲染器中，Blosc 通常作为 OpenVDB 的依赖项，用于体积数据的高效压缩存储。

## 查找逻辑

模块按照以下优先级搜索 Blosc：

1. **用户指定路径**：若定义了 `BLOSC_ROOT_DIR`（CMake 变量或环境变量），则优先在该路径下搜索。
2. **系统默认路径**：`/opt/lib/blosc`

### 头文件搜索

在搜索目录的 `include` 子目录下查找 `blosc.h`。

### 库文件搜索

在搜索目录的 `lib64`、`lib` 子目录下查找名为 `blosc` 的库文件。

### 版本要求

该模块未设置特定的版本要求。

## 输出变量

| 变量名 | 说明 |
|--------|------|
| `BLOSC_FOUND` | 布尔值，若为 `FALSE` 则表示未找到 Blosc |
| `BLOSC_INCLUDE_DIRS` | Blosc 头文件目录，在找到 Blosc 时设置 |
| `BLOSC_LIBRARIES` | 链接 Blosc 所需的库文件列表 |
| `BLOSC_ROOT_DIR` | 搜索 Blosc 的根目录，可通过环境变量设置 |

### 内部变量（高级）

| 变量名 | 说明 |
|--------|------|
| `BLOSC_INCLUDE_DIR` | Blosc 头文件目录（单数形式） |
| `BLOSC_LIBRARY` | Blosc 库文件路径（单数形式） |

## 依赖关系

该模块使用 CMake 内置的 `FindPackageHandleStandardArgs` 模块来处理标准的查找参数。无其他外部 Find 模块依赖。
