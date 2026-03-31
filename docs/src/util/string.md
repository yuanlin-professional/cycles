# string.h / string.cpp - 字符串操作工具函数库

## 概述
本文件提供了 Cycles 渲染器中通用的字符串操作工具函数集合，包括格式化输出、字符串分割/替换/比较、大小写转换、前后缀判断等常用操作。在 Windows 平台上还提供了 UTF-8 与宽字符串（`wstring`）之间的转换函数，以支持非 ASCII 文件名。同时包含将字节大小或数量转换为人类可读格式的函数。

## 类与结构体
本文件未定义类或结构体，仅提供自由函数。使用了 OIIO 的 `string_view` 替代 `std::string_view`（因命名空间冲突暂未切换）。

## 核心函数

### 格式化与转换
| 函数 | 说明 |
|------|------|
| `string string_printf(const char *format, ...)` | 类似 `sprintf` 的安全格式化函数，自动扩展缓冲区（最大 64KB） |
| `string string_from_bool(bool var)` | 将布尔值转换为 `"True"` / `"False"` |
| `string string_hex(const uint8_t *data, size_t size)` | 将字节数组转换为十六进制字符串 |
| `string to_string(const char *str)` | C 字符串转 `std::string` |
| `string to_string(const float4 &v)` | `float4` 转逗号分隔的字符串表示 |
| `string string_to_lower(const string &s)` | 转换为小写字符串 |

### 字符串操作
| 函数 | 说明 |
|------|------|
| `bool string_iequals(const string &a, const string &b)` | 忽略大小写的字符串比较 |
| `void string_split(vector<string> &tokens, const string &str, ...)` | 按分隔符拆分字符串，支持跳过空 token |
| `void string_replace(string &haystack, const string &needle, const string &other)` | 替换所有匹配子串（长度可变） |
| `void string_replace_same_length(string &haystack, ...)` | 等长替换，使用 `memcpy` 更高效 |
| `bool string_startswith(string_view s, string_view start)` | 判断字符串前缀 |
| `bool string_endswith(string_view s, string_view end)` | 判断字符串后缀 |
| `string string_strip(const string &s)` | 去除首尾空格 |
| `string string_remove_trademark(const string &s)` | 移除商标符号 `(TM)` 和 `(R)` |

### Windows 宽字符支持（`#ifdef _WIN32`）
| 函数 | 说明 |
|------|------|
| `wstring string_to_wstring(const string &path)` | UTF-8 转宽字符串（使用 `MultiByteToWideChar`） |
| `string string_from_wstring(const wstring &path)` | 宽字符串转 UTF-8 |
| `string string_to_ansi(const string &str)` | UTF-8 转 ANSI 编码 |

### 人类可读格式
| 函数 | 说明 |
|------|------|
| `string string_human_readable_size(size_t size)` | 将字节数转为如 `"1.50G"` 的可读格式（B/K/M/G/T...） |
| `string string_human_readable_number(size_t num)` | 将数字加千位分隔符，如 `"1,234,567"` |

## 依赖关系
- **内部头文件**: `util/vector.h`, `util/types_float4.h`（cpp）, `util/windows.h`（Windows）
- **外部依赖**: `<OpenImageIO/string_view.h>`
- **被引用**: 几乎所有 `src/util/` 和 `src/` 下的源文件都依赖本文件，是最基础的工具头文件之一

## 关联文件
- `util/vector.h` - 容器定义（`vector`）
- `util/types_float4.h` - `float4` 类型定义
- `util/windows.h` - Windows API 封装
