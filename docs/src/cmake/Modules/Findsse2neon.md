# Findsse2neon.cmake - sse2neon 查找模块

## 概述

该模块用于查找 sse2neon 头文件库。sse2neon 是一个将 Intel SSE（Streaming SIMD Extensions）内联函数（intrinsics）转换为 ARM NEON 内联函数的翻译层头文件。这是一个仅头文件（header-only）的库，不需要链接任何库文件。Cycles 渲染器使用 sse2neon 在 ARM 架构（如 Apple Silicon、Linux ARM64）上运行原本针对 x86 SSE 指令集编写的 SIMD 优化代码。

## 查找逻辑

### 搜索路径

模块按以下优先级搜索 sse2neon：

1. CMake 变量 `SSE2NEON_ROOT_DIR`（若已定义）
2. 环境变量 `SSE2NEON_ROOT_DIR`

### 搜索过程

1. **头文件搜索**：在搜索路径的 `include` 子目录中查找 `sse2neon.h`。

由于 sse2neon 是仅头文件的库，不需要查找库文件。

### 版本要求

模块未指定最低版本要求。

## 输出变量

| 变量名 | 说明 |
|--------|------|
| `SSE2NEON_FOUND` | 布尔值，指示是否成功找到 sse2neon |
| `SSE2NEON_INCLUDE_DIRS` | sse2neon 头文件目录列表 |
| `SSE2NEON_ROOT_DIR` | sse2neon 的搜索基础目录（支持环境变量） |

## 依赖关系

- 无直接 CMake 模块依赖
- 使用 CMake 内置的 `FindPackageHandleStandardArgs` 模块
- 仅在 ARM 架构（如 AArch64）上使用，x86/x86_64 平台直接使用原生 SSE 指令
