# system.h / system.cpp - 系统信息查询接口

## 概述

`system.h` 和 `system.cpp` 提供跨平台的系统信息查询功能，包括 CPU 型号识别、指令集支持检测（SSE4.2、AVX2）、物理内存大小、控制台宽度以及进程 ID 获取。这些信息用于 Cycles 在运行时选择最优的内核实现和资源分配策略。

## 类与结构体

### `CPUCapabilities`（仅 x86/x64 平台，内部结构体）

| 字段 | 说明 |
|------|------|
| `bool sse42` | 是否支持 SSE4.2 指令集 |
| `bool avx2` | 是否支持 AVX2 指令集 |

该结构体通过 `system_cpu_capabilities()` 函数延迟初始化（仅首次调用时检测）。

## 核心函数/宏定义

| 函数 | 说明 |
|------|------|
| `int system_console_width()` | 获取控制台输出宽度（字符数），失败时返回 80 |
| `string system_cpu_brand_string()` | 获取 CPU 型号字符串，失败时返回 "Unknown CPU" |
| `int system_cpu_bits()` | 返回系统位数（32 或 64），基于指针大小计算 |
| `bool system_cpu_support_sse42()` | 检测 CPU 是否支持 SSE4.2（含所有前置指令集） |
| `bool system_cpu_support_avx2()` | 检测 CPU 是否支持 AVX2（含 FMA3、BMI1、BMI2 等） |
| `size_t system_physical_ram()` | 获取物理内存总量（字节） |
| `uint64_t system_self_process_id()` | 获取当前进程 ID |

### 平台实现详情

**`system_console_width()`：**
- Windows：使用 `GetConsoleScreenBufferInfo`
- Unix/macOS：使用 `ioctl(TIOCGWINSZ)`

**`system_cpu_brand_string()`：**
- macOS：通过 `sysctlbyname("machdep.cpu.brand_string")` 获取
- Windows x86/x64：通过 `__cpuid` 内建函数读取扩展 CPUID 信息（0x80000002-0x80000004）
- Windows ARM64：通过注册表 `HKLM\HARDWARE\DESCRIPTION\System\CentralProcessor\0` 读取
- Linux/其他：解析 `/proc/cpuinfo` 文件

**`system_physical_ram()`：**
- Windows：使用 `GlobalMemoryStatusEx`
- macOS：使用 `sysctlbyname("hw.memsize")`
- Linux：使用 `sysconf(_SC_PAGESIZE) * sysconf(_SC_PHYS_PAGES)`

**`system_self_process_id()`：**
- Windows：使用 `GetCurrentProcessId()`
- Unix：使用 `getpid()`

## 依赖关系

- **内部头文件**: `util/string.h`（用于 `string_remove_trademark`）
- **外部依赖**: `<intrin.h>`（Windows x86）, `<sys/sysctl.h>`（macOS）, `<sys/ioctl.h>`（Unix）
- **被引用**: `util/openimagedenoise.h`, `util/guiding.h`, `session/tile.cpp`, `device/metal/device.mm`, `device/cpu/kernel_function.h`, `device/device.cpp`, `test/util_float8_test.h`

## 实现细节

1. **CPUID 跨平台封装**：在非 Windows 的 x86 平台上，手动实现 `__cpuid` 函数，使用内联汇编调用 `cpuid` 指令。32 位 x86 需要额外保存/恢复 `ebx` 寄存器。

2. **AVX2 检测完整性**：`system_cpu_support_avx2()` 不仅检测 AVX2 标志位，还验证操作系统是否支持 XSAVE/XRESTORE 以及 YMM 寄存器保存（通过 `xgetbv` 检查 XCR0 的第 1、2 位），同时要求 FMA3、F16C、BMI1、BMI2 全部可用。

3. **延迟初始化**：CPU 能力检测使用静态局部变量实现单次初始化（`caps_init` 标志），避免重复执行 CPUID 指令。

4. **非 x86 平台回退**：在 ARM 等非 x86 平台上，`system_cpu_support_sse42()` 和 `system_cpu_support_avx2()` 直接返回 `false`。

## 关联文件

- `util/windows.h` - Windows API 头文件预处理
- `util/string.h` - 字符串工具（`string_remove_trademark` 清理品牌字符串中的商标符号）
- `device/cpu/kernel_function.h` - 根据 CPU 能力选择优化内核
