# log.h / log.cpp - 日志系统

## 概述
本文件实现了 Cycles 渲染器的日志系统，提供多级别日志输出、条件检查宏和自定义日志回调功能。日志系统支持 10 个级别（从 FATAL 到 TRACE），默认输出到 stdout/stderr，也可通过 `LogFunction` 回调自定义输出目标。系统使用流式（`<<`）语法记录日志，并通过宏实现惰性求值以避免不必要的性能开销。

## 类与结构体

### `enum LogLevel`
日志级别枚举，从严重到详细依次为：
| 级别 | 值 | 说明 |
|------|---|------|
| `LOG_LEVEL_FATAL` | 0 | 致命错误，程序将调用 `abort()` 终止 |
| `LOG_LEVEL_DFATAL` | 1 | 仅在 Debug 构建中为致命错误 |
| `LOG_LEVEL_ERROR` | 2 | 一般错误 |
| `LOG_LEVEL_DERROR` | 3 | 仅 Debug 构建中的错误 |
| `LOG_LEVEL_WARNING` | 4 | 警告 |
| `LOG_LEVEL_DWARNING` | 5 | 仅 Debug 构建中的警告 |
| `LOG_LEVEL_INFO_IMPORTANT` | 6 | 默认打印的重要信息 |
| `LOG_LEVEL_INFO` | 7 | 设备、场景和特性相关信息 |
| `LOG_LEVEL_DEBUG` | 8 | 工作执行和计时/内存统计 |
| `LOG_LEVEL_TRACE` | 9 | 详细的代码执行追踪 |

### `class LogMessage`
日志消息构造辅助类。构造时记录级别、文件位置和函数名；通过 `stream()` 方法返回 `std::stringstream` 供流式写入；析构时调用 `_log_message()` 完成实际输出。

### `LogFunction`（类型别名）
自定义日志回调函数签名：`void (*)(LogLevel, const char* file_line, const char* func, const char* msg)`

## 核心函数

### 配置 API
| 函数 | 说明 |
|------|------|
| `void log_init(LogFunction func)` | 初始化日志系统，可设置自定义回调（默认输出到 stdout/stderr） |
| `void log_level_set(LogLevel level)` | 设置当前日志级别 |
| `void log_level_set(const string &level)` | 通过字符串设置日志级别（如 `"debug"`、`"trace"`） |
| `const char *log_level_to_string(LogLevel level)` | 日志级别转字符串 |
| `LogLevel log_string_to_level(const string &str)` | 字符串转日志级别（不区分大小写） |

### 日志宏
| 宏 | 说明 |
|------|------|
| `LOG(level)` / `LOG_IF(level, condition)` | 按级别记录日志，支持条件判断，惰性求值 |
| `LOG_FATAL` ... `LOG_TRACE` | 各级别的快捷宏 |
| `LOG_IS_ON(level)` | 检查指定级别日志是否启用 |
| `CHECK(expression)` | 断言表达式为真，失败时触发 FATAL |
| `CHECK_GE/NE/EQ/GT/LT/LE(a, b)` | 比较断言 |
| `DCHECK(expression)` / `DCHECK_*` | 仅 Debug 构建中生效的断言 |

### 流输出运算符
为 `int2`、`float3`、`float4` 类型重载了 `operator<<`，方便日志输出。

## 依赖关系
- **内部头文件**: `util/defines.h`, `util/string.h`, `util/types.h`, `util/math.h`（cpp）, `util/time.h`（cpp）
- **被引用**: 几乎所有 Cycles 源文件都使用本日志系统

## 关联文件
- `util/string.h` - `string_split`、`string_to_lower` 用于日志处理
- `util/time.h` - `time_dt()` 用于时间戳记录
