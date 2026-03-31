# FindLLVM.cmake - LLVM 查找模块

## 概述

该模块用于查找 LLVM（Low Level Virtual Machine）编译器基础设施库。LLVM 是一个模块化的编译器和工具链技术集合，提供代码优化和代码生成功能。在 Cycles 渲染器中，LLVM 用于运行时的着色器内核编译和 JIT（即时编译）优化，特别是为 OSL（Open Shading Language）提供后端支持。

## 查找逻辑

该模块主要通过 `llvm-config` 工具来定位 LLVM 安装：

### llvm-config 查找

1. **指定根目录时**：若定义了 `LLVM_ROOT_DIR`，在 `${LLVM_ROOT_DIR}/bin` 下查找 `llvm-config`。
   - 若定义了 `LLVM_VERSION`，优先查找 `llvm-config-${LLVM_VERSION}`。
2. **未指定根目录时**：在系统 PATH 中查找 `llvm-config`。
   - 同样优先查找版本化的 `llvm-config-${LLVM_VERSION}`。

### 信息提取

通过 `llvm-config` 获取以下信息：

- `--includedir`：头文件目录
- `--version`：LLVM 版本（若未预定义 `LLVM_VERSION`）
- `--prefix`：安装前缀（若未预定义 `LLVM_ROOT_DIR`）
- `--libdir`：库文件路径

### 库文件搜索

- **动态库模式**（默认）：查找 `LLVM-${LLVM_VERSION}` 共享库，若未找到则回退到查找 `LLVMAnalysis` 静态库。
- **静态库模式**（`LLVM_STATIC` 为 `TRUE`）：查找 `LLVMAnalysis` 静态库，找到后通过 `llvm-config --libfiles` 获取完整的静态库列表。

### 版本要求

可通过 `LLVM_VERSION` 变量指定所需的 LLVM 版本。若未指定，模块会自动从 `llvm-config --version` 获取。

## 输出变量

| 变量名 | 说明 |
|--------|------|
| `LLVM_FOUND` | 布尔值，若为 `FALSE` 则表示未找到 LLVM |
| `LLVM_INCLUDE_DIRS` | LLVM 头文件目录 |
| `LLVM_LIBRARIES` | 链接 LLVM 所需的库文件列表 |
| `LLVM_ROOT_DIR` | LLVM 安装根目录（缓存变量） |
| `LLVM_VERSION` | LLVM 版本字符串（缓存变量） |
| `LLVM_LIBPATH` | LLVM 库文件目录路径（缓存变量） |

### 内部变量（高级）

| 变量名 | 说明 |
|--------|------|
| `LLVM_LIBRARY` | LLVM 库文件路径（单数形式，静态模式下为分号分隔的列表） |
| `LLVM_CONFIG` | `llvm-config` 可执行文件路径 |

## 依赖关系

该模块依赖系统上安装的 `llvm-config` 工具程序来定位 LLVM。使用 CMake 内置的 `FindPackageHandleStandardArgs` 模块。

> **相关模块**：`FindClang.cmake` 依赖该模块提供的 LLVM 路径信息来查找 Clang 编译器库。
