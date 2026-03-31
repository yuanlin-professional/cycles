# path.h / path.cpp - 文件路径与 I/O 操作工具库

## 概述
本文件提供 Cycles 渲染器中的文件系统路径操作和文件 I/O 工具函数。包括程序路径管理、路径字符串拼接与拆分、文件信息查询、目录创建、二进制/文本文件读写（支持 Zstandard 压缩解压）、源代码预处理器 `#include` 替换，以及编译内核的 LRU 缓存管理。跨平台支持 Windows（含 UNC 路径处理）和 Unix 系统。

## 类与结构体

### `directory_iterator`（匿名命名空间，内部类）
跨平台目录遍历迭代器。Windows 版基于 `FindFirstFileW`/`FindNextFileW`，Unix 版基于 `scandir`/`alphasort`。自动跳过 `.` 和 `..` 目录项。

### `SourceReplaceState`
源代码 `#include` 替换的状态管理结构体：
- `base` - 所有相对路径 include 的基目录
- `processed_files` - 已处理文件的缓存映射（避免重复处理）
- `pragma_onced` - 记录已包含的 `#pragma once` 文件集合

## 核心函数

### 程序路径管理
| 函数 | 说明 |
|------|------|
| `void path_init(const string &path, const string &user_path)` | 初始化程序路径和用户路径 |
| `string path_get(const string &sub)` | 获取程序相对路径，支持环境变量 `CYCLES_SHADER_PATH` 和 `CYCLES_KERNEL_PATH` |
| `string path_user_get(const string &sub)` | 获取用户数据路径 |
| `string path_cache_get(const string &sub)` | 获取缓存路径（Linux/macOS 使用 XDG_CACHE_HOME） |

### 路径字符串操作
| 函数 | 说明 |
|------|------|
| `string path_filename(const string &path)` | 提取文件名部分 |
| `string path_dirname(const string &path)` | 提取目录部分 |
| `string path_join(const string &dir, const string &file)` | 拼接目录和文件名 |
| `string path_escape(const string &path)` | 转义路径中的空格 |
| `bool path_is_relative(const string &path)` | 判断是否为相对路径 |

### 文件信息查询
| 函数 | 说明 |
|------|------|
| `size_t path_file_size(const string &path)` | 获取文件大小 |
| `bool path_exists(const string &path)` | 判断路径是否存在 |
| `bool path_is_directory(const string &path)` | 判断是否为目录 |
| `string path_files_md5_hash(const string &dir)` | 递归计算目录下所有文件的 MD5 哈希 |
| `uint64_t path_modified_time(const string &path)` | 获取文件最后修改时间 |

### 文件读写
| 函数 | 说明 |
|------|------|
| `FILE *path_fopen(const string &path, const string &mode)` | 跨平台文件打开（Windows 使用 `_wfopen`） |
| `bool path_write_binary(const string &path, const vector<uint8_t> &binary)` | 写入二进制文件 |
| `bool path_read_binary(const string &path, vector<uint8_t> &binary)` | 读取二进制文件到内存 |
| `bool path_read_compressed_binary(const string &path, vector<uint8_t> &binary)` | 读取并解压 `.zst` 压缩的二进制文件 |
| `bool path_write_text` / `path_read_text` / `path_read_compressed_text` | 文本文件对应版本 |

### 源码预处理
| 函数 | 说明 |
|------|------|
| `string path_source_replace_includes(const string &source, const string &path)` | 内置的简易 C 预处理器，将 `#include "..."` 替换为文件内容，处理 `#pragma once`，生成 `#line` 指令 |

### 内核缓存（LRU）
| 函数 | 说明 |
|------|------|
| `bool path_cache_kernel_exists_and_mark_used(const string &path)` | 检查内核缓存是否存在并更新访问时间 |
| `void path_cache_kernel_mark_added_and_clear_old(const string &path, size_t max)` | 注册新内核并清理同目录下旧缓存（默认保留 5 个） |

## 依赖关系
- **内部头文件**: `util/string.h`, `util/vector.h`, `util/algorithm.h`, `util/map.h`, `util/md5.h`, `util/set.h`, `util/windows.h`
- **外部依赖**: `<OpenImageIO/filesystem.h>`, `<OpenImageIO/strutil.h>`, `<OpenImageIO/sysutil.h>`, `<zstd.h>`
- **被引用**: `scene/image.cpp`, `device/` 下的内核编译模块, `app/` 下的应用程序入口

## 关联文件
- `util/string.h` - 字符串工具函数
- `util/md5.h` - MD5 哈希计算
- `util/windows.h` - Windows API 封装
