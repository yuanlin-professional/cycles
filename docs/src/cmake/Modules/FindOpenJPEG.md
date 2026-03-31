# FindOpenJPEG.cmake - OpenJPEG 查找模块

## 概述

该模块用于查找 OpenJPEG 库。OpenJPEG 是 JPEG 2000 图像压缩标准的开源编解码器实现，支持 JPEG 2000 格式的编码和解码操作。Cycles 渲染器通过该库支持 JPEG 2000 格式的纹理和图像文件。

## 查找逻辑

### 搜索路径

模块按以下优先级搜索 OpenJPEG：

1. CMake 变量 `OPENJPEG_ROOT_DIR`（若已定义）
2. 环境变量 `OPENJPEG_ROOT_DIR`

### 搜索过程

1. **头文件搜索**：在搜索路径的 `include` 子目录中查找 `openjpeg.h`。支持多个版本化的子目录后缀，从 `openjpeg-2.9` 向下兼容至 `openjpeg-2.0`。
2. **库文件搜索**：在 `lib64` 和 `lib` 子目录中查找名为 `openjp2` 的库文件。

### 版本要求

模块未强制指定版本要求，但通过 PATH_SUFFIXES 支持 OpenJPEG 2.0 至 2.9 的版本范围。搜索优先级从高版本到低版本依次递减。

## 输出变量

| 变量名 | 说明 |
|--------|------|
| `OPENJPEG_FOUND` | 布尔值，指示是否成功找到 OpenJPEG |
| `OPENJPEG_INCLUDE_DIRS` | OpenJPEG 头文件目录列表 |
| `OPENJPEG_LIBRARIES` | 需要链接的 OpenJPEG 库列表 |
| `OPENJPEG_ROOT_DIR` | OpenJPEG 的搜索基础目录（支持环境变量） |

### 内部变量（非公开接口）

| 变量名 | 说明 |
|--------|------|
| `OPENJPEG_LIBRARY` | OpenJPEG 库文件路径 |

## 依赖关系

- 无直接 CMake 模块依赖
- 使用 CMake 内置的 `FindPackageHandleStandardArgs` 模块
