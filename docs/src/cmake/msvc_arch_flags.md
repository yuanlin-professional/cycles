# msvc_arch_flags.c - MSVC CPU 架构标志检测工具

## 概述

该文件是一个独立的 C 语言小程序，用于在 Windows MSVC 环境下运行时检测当前 CPU 支持的最高 SIMD 指令集级别，并输出对应的 MSVC 编译器架构标志（`/arch:AVX2` 或 `/arch:AVX`）。该工具可在构建系统中被编译并执行，以便自动选择最佳的架构优化选项。

## 核心逻辑

### 全局变量

- `__isa_available`：由 Microsoft C 运行时 (CRT) 定义的外部变量，表示当前 CPU 可用的最高指令集架构级别

### 函数：`get_arch_flags()`

根据 `__isa_available` 的值返回对应的 MSVC 编译器架构标志：

| 检测条件 | 返回值 | 说明 |
|----------|--------|------|
| `__isa_available >= __ISA_AVAILABLE_AVX2` | `/arch:AVX2` | CPU 支持 AVX2 指令集 |
| `__isa_available >= __ISA_AVAILABLE_AVX` | `/arch:AVX` | CPU 支持 AVX 指令集 |
| 其他情况 | `""` | 不添加额外架构标志（使用默认 SSE2） |

检测优先级：AVX2 > AVX > 默认（SSE2）

### 函数：`main()`

程序入口，调用 `get_arch_flags()` 并将结果输出到标准输出，供构建脚本捕获使用。

## 依赖关系

### 系统头文件

- `<isa_availability.h>` - Microsoft 专用头文件，定义 `__ISA_AVAILABLE_AVX`、`__ISA_AVAILABLE_AVX2` 等常量
- `<stdio.h>` - 标准输入输出

### 运行时依赖

- Microsoft CRT（提供 `__isa_available` 全局变量）
- 仅适用于 Windows MSVC 编译环境
