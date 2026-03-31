# util_path_test.cpp - 文件路径工具函数测试

## 概述

此文件测试 Cycles 工具库中的文件路径操作函数，包括文件名提取、目录名提取、路径拼接、路径转义和相对路径判断。由于路径分隔符在 Unix 和 Windows 平台上不同，测试通过条件编译分别覆盖两个平台的行为。

## 测试用例

### path_filename 测试组 - 文件名提取

#### 跨平台测试
| 测试名 | 输入 | 期望输出 |
|--------|------|----------|
| `file_only` | `"foo.txt"` | `"foo.txt"` |
| `empty` | `""` | `""` |

#### Unix 平台测试（`#ifndef _WIN32`）
| 测试名 | 输入 | 期望输出 |
|--------|------|----------|
| `simple_unix` | `"/tmp/foo.txt"` | `"foo.txt"` |
| `root_unix` | `"/"` | `"/"` |
| `last_slash_unix` | `"/tmp/foo.txt/"` | `"."` |
| `alternate_slash_unix` | `"/tmp\\foo.txt"` | `"tmp\\foo.txt"` |

#### Windows 平台测试（`#ifdef _WIN32`）
| 测试名 | 输入 | 期望输出 |
|--------|------|----------|
| `simple_windows` | `"C:\\tmp\\foo.txt"` | `"foo.txt"` |
| `root_windows` | `"C:\\"` | `"\\"` |
| `last_slash_windows` | `"C:\\tmp\\foo.txt\\"` | `"."` |
| `alternate_slash_windows` | `"C:\\tmp/foo.txt"` | `"foo.txt"` |

### path_dirname 测试组 - 目录名提取

#### 跨平台测试
| 测试名 | 输入 | 期望输出 |
|--------|------|----------|
| `file_only` | `"foo.txt"` | `""` |
| `empty` | `""` | `""` |

#### Unix 平台测试
| 测试名 | 输入 | 期望输出 |
|--------|------|----------|
| `simple_unix` | `"/tmp/foo.txt"` | `"/tmp"` |
| `root_unix` | `"/"` | `""` |
| `last_slash_unix` | `"/tmp/foo.txt/"` | `"/tmp/foo.txt"` |
| `alternate_slash_unix` | `"/tmp\\foo.txt"` | `"/"` |

#### Windows 平台测试
| 测试名 | 输入 | 期望输出 |
|--------|------|----------|
| `simple_windows` | `"C:\\tmp\\foo.txt"` | `"C:\\tmp"` |
| `root_windows` | `"C:\\"` | `"C:"` |
| `last_slash_windows` | `"C:\\tmp\\foo.txt\\"` | `"C:\\tmp\\foo.txt"` |
| `alternate_slash_windows` | `"C:\\tmp/foo.txt"` | `"C:\\tmp"` |

### path_join 测试组 - 路径拼接

#### 跨平台测试
| 测试名 | 目录 | 文件名 | 期望输出 |
|--------|------|--------|----------|
| `empty_both` | `""` | `""` | `""` |
| `empty_directory` | `""` | `"foo.txt"` | `"foo.txt"` |
| `empty_filename` | `"foo"` | `""` | `"foo"` |

#### 平台相关测试
- Unix 平台: 测试正斜杠和反斜杠混合场景（共 11 个测试用例）
- Windows 平台: 测试反斜杠和正斜杠混合场景（共 11 个测试用例）

### path_escape 测试组 - 路径转义
| 测试名 | 输入 | 期望输出 | 说明 |
|--------|------|----------|------|
| `no_escape_chars` | `"/tmp/foo/bar"` | `"/tmp/foo/bar"` | 无需转义 |
| `simple` | `"/tmp/foo bar"` | `"/tmp/foo\\ bar"` | 单个空格转义 |
| `simple_end` | `"/tmp/foo/bar "` | `"/tmp/foo/bar\\ "` | 末尾空格 |
| `multiple` | `"/tmp/foo  bar"` | `"/tmp/foo\\ \\ bar"` | 多个空格 |
| `simple_multiple_end` | `"/tmp/foo/bar  "` | `"/tmp/foo/bar\\ \\ "` | 末尾多空格 |

### path_is_relative 测试组 - 相对路径判断

#### 跨平台测试
| 测试名 | 输入 | 期望结果 |
|--------|------|----------|
| `filename` | `"foo.txt"` | `true` |

#### Unix 平台测试
- 绝对路径 `"/tmp/foo.txt"` -> `false`
- 相对目录 `"tmp/foo.txt"` -> `true`
- Windows 风格路径在 Unix 上被视为相对路径

#### Windows 平台测试
- 绝对路径 `"C:\\tmp\\foo.txt"` -> `false`
- 相对目录 `"tmp\\foo.txt"` -> `true`
- Unix 风格绝对路径在 Windows 上被视为相对路径

## 依赖关系
- **被测源文件**: `util/path.h`
- **测试框架**: Google Test (GTest)

## 关联文件
- `src/util/path.h` - 路径工具函数声明
- `src/util/path.cpp` - 路径工具函数实现
