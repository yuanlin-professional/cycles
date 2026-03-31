# types_base.h - 基础类型定义与内存对齐工具函数

## 概述

`types_base.h` 是 Cycles 渲染器类型系统的根基文件，定义了整个内核中使用的基本无符号类型别名（`uchar`、`uint`、`ushort`）、设备指针类型 `device_ptr`，以及一系列常用的内存对齐与整数运算工具函数。该文件还负责在非 GPU 环境下引入 SIMD 优化和标准整数类型头文件，并提供跨平台的调试打印支持。

## 类与结构体

本文件不定义结构体或类，主要通过 `using` 声明提供类型别名：

| 类型别名 | 原始类型 | 说明 |
|----------|----------|------|
| `uchar` | `unsigned char` | 无符号字符类型简写 |
| `uint` | `unsigned int` | 无符号整数类型简写 |
| `ushort` | `unsigned short` | 无符号短整数类型简写 |
| `device_ptr` | `uint64_t` | 通用设备内存指针（仅 CPU 端） |

## 核心函数/运算符重载

| 函数签名 | 功能说明 |
|----------|----------|
| `size_t align_up(size_t offset, size_t alignment)` | 将 `offset` 向上对齐到 `alignment` 的整数倍（要求 `alignment` 为 2 的幂） |
| `size_t divide_up(size_t x, size_t y)` | 向上整除，返回 `ceil(x / y)` |
| `size_t round_up(size_t x, size_t multiple)` | 将 `x` 向上取整到 `multiple` 的整数倍 |
| `size_t round_down(size_t x, size_t multiple)` | 将 `x` 向下取整到 `multiple` 的整数倍 |
| `bool is_power_of_two(size_t x)` | 判断 `x` 是否为 2 的幂 |
| `void print_float(const char *label, float a)` | 调试用浮点数打印，支持 CUDA/Metal 等多后端 |

## 依赖关系

- **内部头文件**:
  - `util/defines.h` — 宏定义（`ccl_device_inline`、`CCL_NAMESPACE_BEGIN` 等）
  - `util/optimization.h` — CPU 优化选项（仅非 GPU）
  - `util/simd.h` — SIMD 指令集封装（仅非 GPU）
- **标准库**: `<cstdlib>`、`<cstdint>`、`<cstdio>`（按平台条件引入）
- **被引用**: 几乎所有 `types_*.h` 文件、`util/types.h`、`util/math_base.h`、`kernel/util/nanovdb.h`、`scene/light_tree_debug.h`

## 关联文件

- `src/util/types.h` — 统一引入所有类型头文件的聚合头文件
- `src/util/defines.h` — 跨平台宏定义
- `src/util/simd.h` — SSE/AVX SIMD 类型与内联函数
- `src/util/optimization.h` — 编译优化相关配置
