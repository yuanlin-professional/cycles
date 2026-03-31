# configure_build.cmake - 全局构建配置与编译器设置

## 概述

该文件负责配置 Cycles 渲染器的全局 CMake 构建设置，包括输出目录配置、C++ 标准设定以及各编译器平台（GCC、Clang、Apple/macOS、MSVC）的编译和链接标志。它是构建系统初始化阶段的核心配置文件。

## 构建选项 / 变量

### 输出目录

| 变量名 | 值 | 说明 |
|--------|-----|------|
| `CMAKE_ARCHIVE_OUTPUT_DIRECTORY` | `${CMAKE_BINARY_DIR}/lib` | 静态库输出目录 |
| `CMAKE_LIBRARY_OUTPUT_DIRECTORY` | `${CMAKE_BINARY_DIR}/lib` | 共享库输出目录 |
| `CMAKE_RUNTIME_OUTPUT_DIRECTORY` | `${CMAKE_BINARY_DIR}/bin` | 可执行文件输出目录 |

### C++ 标准

| 变量名 | 值 | 说明 |
|--------|-----|------|
| `CMAKE_CXX_STANDARD` | `17` | 使用 C++17 标准 |
| `CMAKE_CXX_STANDARD_REQUIRED` | `ON` | 强制要求 C++17 |
| `CMAKE_CXX_EXTENSIONS` | `OFF` | 禁用编译器扩展 |

### 平台链接库

| 变量名 | 说明 |
|--------|------|
| `PLATFORM_LINKLIBS` | 平台相关链接库列表（MSVC 下包含 psapi、Version、Dbghelp、Shlwapi） |

## 关键逻辑

### GCC / Clang 配置

两者共享相同的编译标志策略：

- **C 标志**：`-Wall -Wno-sign-compare -fno-strict-aliasing -fPIC`
- **C++ 标志**：在 C 标志基础上增加 `-Wno-invalid-offsetof -std=c++17`

### macOS (Apple) 配置

- 非 Xcode 生成器时，手动设置 `-mmacosx-version-min` 以强制最低部署目标版本
- C++ 标志额外添加 `-stdlib=libc++`
- 定义宏 `MACOSX_DEPLOYMENT_TARGET`
- 链接器标志中添加 `-Xlinker -no_warn_duplicate_libraries` 以抑制新版链接器的重复库警告

### MSVC 配置

**C++ 编译标志**：`/nologo /J /Gd /EHsc /bigobj /MP /std:c++17`

| 标志 | 说明 |
|------|------|
| `/nologo` | 抑制版权信息输出 |
| `/J` | 默认 char 类型为 unsigned |
| `/Gd` | 使用 `__cdecl` 调用约定 |
| `/EHsc` | 启用 C++ 异常处理 |
| `/bigobj` | 支持大目标文件 |
| `/MP` | 启用多处理器并行编译 |

**构建类型特定标志**：

| 构建类型 | 标志 | 说明 |
|----------|------|------|
| Debug (64位) | `/Od /RTC1 /MDd /Zi` | 无优化 + 运行时检查 + 动态调试CRT + 调试信息 |
| Debug (32位) | `/Od /RTC1 /MDd /ZI` | 同上，但使用编辑并继续调试信息 |
| Release | `/O2 /Ob2 /MD` | 最大速度优化 + 内联扩展 + 动态CRT |
| MinSizeRel | `/O1 /Ob1 /MD` | 最小大小优化 |
| RelWithDebInfo | `/O2 /Ob1 /MD /Zi` | 速度优化 + 调试信息 |

**MSVC 特殊处理**：

- MSVC 2019+ (版本 >= 1920)：禁用 `/Zc:inline` 以避免 USD 的 `TF_REGISTRY_FUNCTION` 宏符号被剥离的问题
- 添加 `/wd4996` 抑制弃用警告
- 添加 `/Zc:__cplusplus` 使 `__cplusplus` 宏报告正确的 C++ 版本
- 定义 `_USE_MATH_DEFINES` 以启用数学常量（如 `M_PI`）

## 依赖关系

- 无外部 cmake 文件依赖
- 所有设置均使用 CMake 内置变量和命令
