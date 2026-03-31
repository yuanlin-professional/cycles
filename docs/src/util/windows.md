# windows.h / windows.cpp - Windows 平台头文件安全封装

## 概述

`windows.h` 和 `windows.cpp` 提供 Windows 平台 `<windows.h>` 的安全封装，确保在包含系统头文件前定义必要的精简化宏（`WIN32_LEAN_AND_MEAN`、`NOMINMAX`、`NOGDI`），避免与 Cycles 代码产生符号冲突。同时提供 Windows 版本检测函数。

## 类与结构体

无自定义类或结构体。

## 核心函数/宏定义

### 预处理宏

| 宏 | 说明 |
|----|------|
| `NOGDI` | 排除 GDI 相关定义，避免 `min`/`max` 等符号污染 |
| `NOMINMAX` | 阻止 Windows 定义 `min()` 和 `max()` 宏，避免与 `std::min`/`std::max` 冲突 |
| `WIN32_LEAN_AND_MEAN` | 精简 `<windows.h>`，排除不常用的 API（加速编译） |

这三个宏仅在未被预定义时才定义，使用 `#ifndef` 保护。

### 函数

| 函数 | 说明 |
|------|------|
| `bool system_windows_version_at_least(int major, int build)` | 检测当前 Windows 版本是否 >= 指定的主版本号和构建号 |

## 依赖关系

- **内部头文件**: 无
- **外部依赖**: `<windows.h>`（仅 `_WIN32` 平台）
- **被引用**: `util/thread.h`（Windows 平台条件包含）, `util/tbb.h`（Windows 平台条件包含）, `util/simd.h`（MinGW64 条件包含）, `util/system.cpp`, `util/thread.cpp`

## 实现细节

1. **版本检测方法**：`system_windows_version_at_least()` 使用 `ntdll.dll` 中的 `RtlGetVersion` 函数而非已废弃的 `GetVersionEx`。`GetVersionEx` 在 Windows 8.1+ 需要应用程序清单才能返回正确版本号，而 `RtlGetVersion` 始终返回真实版本信息。

2. **动态加载**：通过 `GetModuleHandleW` + `GetProcAddress` 动态获取 `RtlGetVersion` 函数指针，避免链接时依赖 ntdll 导出。

3. **非 Windows 平台**：在非 Windows 平台上，`system_windows_version_at_least()` 始终返回 `false`，参数被 `(void)` 消除未使用警告。

4. **全局影响**：此文件是 Cycles 中所有需要 Windows API 的模块的统一入口。通过确保 `WIN32_LEAN_AND_MEAN` 等宏在 `<windows.h>` 包含之前定义，防止其他库（如 TBB）意外引入完整的 Windows 头文件。

## 关联文件

- `util/tbb.h` - 在 Windows 上优先包含本文件以预处理宏定义
- `util/thread.h` - Windows 平台条件包含本文件
- `util/system.cpp` - 使用 Windows API 查询系统信息
