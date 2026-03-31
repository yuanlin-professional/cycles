# FindOpenColorIO.cmake - OpenColorIO 查找模块

## 概述

该模块用于查找 OpenColorIO（OCIO）库。OpenColorIO 是由 Sony Pictures Imageworks 开发的开源颜色管理框架，提供完整的颜色空间转换和显示管理功能。在 Cycles 渲染器中，OpenColorIO 用于确保渲染管线中颜色的准确性和一致性，支持各种行业标准颜色空间（如 ACES、sRGB 等）的转换。

## 查找逻辑

模块按照以下优先级搜索 OpenColorIO：

1. **用户指定路径**：若定义了 `OPENCOLORIO_ROOT_DIR`（CMake 变量或环境变量），则优先在该路径下搜索。
2. **系统默认路径**：`/opt/lib/ocio`

### 头文件搜索

在搜索目录的 `include` 子目录下查找 `OpenColorIO/OpenColorIO.h`。

### 库文件搜索

在搜索目录的 `lib64`、`lib`、`lib64/static`、`lib/static` 子目录下查找名为 `OpenColorIO` 的库文件。

### 版本检测

从 `OpenColorIO/OpenColorABI.h` 头文件中提取版本信息。模块会进行两次搜索以兼容不同版本：

1. 首先查找 `OCIO_VERSION_STR` 宏（OCIO 2.x）
2. 若失败，查找 `OCIO_VERSION` 宏（OCIO 1.x）

## 输出变量

| 变量名 | 说明 |
|--------|------|
| `OPENCOLORIO_FOUND` | 布尔值，若为 `FALSE` 则表示未找到 OpenColorIO |
| `OPENCOLORIO_INCLUDE_DIRS` | OpenColorIO 头文件目录 |
| `OPENCOLORIO_LIBRARIES` | 链接 OpenColorIO 所需的库文件列表 |
| `OPENCOLORIO_ROOT_DIR` | 搜索 OpenColorIO 的根目录，可通过环境变量设置 |
| `OPENCOLORIO_VERSION` | OpenColorIO 版本字符串 |

### 内部变量（高级）

| 变量名 | 说明 |
|--------|------|
| `OPENCOLORIO_INCLUDE_DIR` | OpenColorIO 头文件目录（单数形式） |
| `OPENCOLORIO_LIBRARY` | OpenColorIO 库文件路径（单数形式） |
| `OPENCOLORIO_OPENCOLORIO_LIBRARY` | OpenColorIO 主库路径 |

## 依赖关系

该模块使用 CMake 内置的 `FindPackageHandleStandardArgs` 模块来处理标准的查找参数。无其他外部 Find 模块依赖。
