# FindZstd.cmake - Zstandard 查找模块

## 概述

该模块用于查找 Zstd（Zstandard）库。Zstandard 是由 Facebook 开发的快速无损数据压缩算法，提供接近 LZ4 的压缩速度和接近 zlib 的压缩比。Cycles 渲染器使用 Zstd 进行高效的数据压缩和解压缩操作，特别是在 OpenVDB 体积数据和场景文件的处理中。

## 查找逻辑

### 搜索路径

模块按以下优先级搜索 Zstd：

1. CMake 变量 `ZSTD_ROOT_DIR`（若已定义）
2. 环境变量 `ZSTD_ROOT_DIR`

### 搜索过程

1. **头文件搜索**：在搜索路径的 `include` 子目录中查找 `zstd.h`。
2. **库文件搜索**：在 `lib64` 和 `lib` 子目录中查找名为 `zstd` 的库文件。

### 版本要求

模块未指定最低版本要求。

## 输出变量

| 变量名 | 说明 |
|--------|------|
| `ZSTD_FOUND` | 布尔值，指示是否成功找到 Zstd |
| `ZSTD_INCLUDE_DIRS` | Zstd 头文件目录列表 |
| `ZSTD_LIBRARIES` | 需要链接的 Zstd 库列表 |
| `ZSTD_ROOT_DIR` | Zstd 的搜索基础目录（支持环境变量） |

### 内部变量（非公开接口）

| 变量名 | 说明 |
|--------|------|
| `ZSTD_LIBRARY` | Zstd 库文件路径 |

## 依赖关系

- 无直接 CMake 模块依赖
- 使用 CMake 内置的 `FindPackageHandleStandardArgs` 模块
