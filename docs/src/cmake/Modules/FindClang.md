# FindClang.cmake - Clang 查找模块

## 概述

该模块用于查找 Clang 编译器库。Clang 是基于 LLVM 的 C/C++/Objective-C 编译器前端。在 Cycles 渲染器中，Clang 用于运行时内核编译和代码优化，特别是在 OSL（Open Shading Language）等着色器编译流程中。

## 查找逻辑

模块按照以下优先级搜索 Clang：

1. **用户指定路径**：若定义了 `CLANG_ROOT_DIR`（CMake 变量或环境变量），则优先在该路径下搜索。
2. **LLVM 路径推导**：若未设置 `LLVM_ROOT_DIR`，模块会通过 `llvm-config` 工具（优先使用版本化的 `llvm-config-${LLVM_VERSION}`）获取 LLVM 安装前缀，作为 Clang 的搜索路径。
3. **系统默认路径**：`/opt/lib/clang`

### 头文件搜索

在搜索目录的 `include` 或 `include/clang` 子目录下查找 `AST/AST.h`。

### 库文件搜索

该模块搜索大量 Clang 组件库，包括但不限于：

- `clangFrontend`、`clangDriver`、`clangSerialization`
- `clangCodeGen`、`clangParse`、`clangSema`
- `clangAST`、`clangLex`、`clangBasic`
- `clangAnalysis`、`clangEdit`、`clangRewrite`
- `clangTooling`、`clangFormat`
- `clangStaticAnalyzerCore`、`clangStaticAnalyzerCheckers`
- 以及更多组件（共约 30 个组件库）

所有组件在搜索目录的 `lib64`、`lib` 子目录下查找。

### 版本要求

该模块本身不强制版本要求，但可通过 `LLVM_VERSION` 变量指定对应的 LLVM/Clang 版本。

## 输出变量

| 变量名 | 说明 |
|--------|------|
| `CLANG_FOUND` | 布尔值，若为 `FALSE` 则表示未找到 Clang |
| `CLANG_INCLUDE_DIRS` | Clang 头文件目录，在 `CLANG_INCLUDE_DIR` 被找到时设置 |
| `CLANG_LIBRARIES` | 链接 Clang 所需的全部组件库文件列表 |
| `CLANG_ROOT_DIR` | 搜索 Clang 的根目录，可通过环境变量设置 |

### 内部变量（高级）

| 变量名 | 说明 |
|--------|------|
| `CLANG_INCLUDE_DIR` | Clang 头文件目录（单数形式） |
| `CLANG_${UPPERCOMPONENT}_LIBRARY` | 各 Clang 组件库的路径（每个组件一个变量） |

## 依赖关系

- **LLVM**：该模块依赖 LLVM 安装。若未指定 `LLVM_ROOT_DIR`，会通过 `llvm-config` 程序自动检测 LLVM 位置。
- 使用 CMake 内置的 `FindPackageHandleStandardArgs` 模块。
