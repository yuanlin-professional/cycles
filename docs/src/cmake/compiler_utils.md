# compiler_utils.cmake - 编译器工具函数与安装宏

## 概述

该文件提供了一组用于编译器标志检测、文件延迟安装以及编译器标志移除的工具函数和宏。这些工具在 Cycles 的整个构建系统中被广泛使用，用于确保编译器兼容性和正确的安装流程。

## 构建选项 / 变量

该文件本身不定义 CMake 缓存选项，但操作以下 CMake 内置变量：

| 变量名 | 说明 |
|--------|------|
| `CMAKE_C_FLAGS` | C 编译器标志（各构建类型） |
| `CMAKE_CXX_FLAGS` | C++ 编译器标志（各构建类型） |
| `CMAKE_C_FLAGS_DEBUG` | Debug 构建的 C 编译器标志 |
| `CMAKE_CXX_FLAGS_DEBUG` | Debug 构建的 C++ 编译器标志 |
| `CMAKE_C_FLAGS_RELEASE` | Release 构建的 C 编译器标志 |
| `CMAKE_CXX_FLAGS_RELEASE` | Release 构建的 C++ 编译器标志 |
| `CMAKE_C_FLAGS_MINSIZEREL` | MinSizeRel 构建的 C 编译器标志 |
| `CMAKE_CXX_FLAGS_MINSIZEREL` | MinSizeRel 构建的 C++ 编译器标志 |
| `CMAKE_C_FLAGS_RELWITHDEBINFO` | RelWithDebInfo 构建的 C 编译器标志 |
| `CMAKE_CXX_FLAGS_RELWITHDEBINFO` | RelWithDebInfo 构建的 C++ 编译器标志 |

全局属性：

| 属性名 | 说明 |
|--------|------|
| `DELAYED_INSTALL_FILES` | 延迟安装的文件列表 |
| `DELAYED_INSTALL_DESTINATIONS` | 延迟安装的目标路径列表 |

## 关键逻辑

### 函数：`add_check_cxx_compiler_flag_impl`

内部实现函数，检测单个 C++ 编译器标志是否受支持：

- 使用 CMake 内置的 `CheckCXXCompilerFlag` 模块进行检测
- 如果标志受支持，将其追加到指定的 CXXFLAGS 变量中
- 如果标志不受支持，输出状态消息提示

### 函数：`ADD_CHECK_CXX_COMPILER_FLAGS`

批量检测编译器标志的公共接口函数：

- 接受一个 CXXFLAGS 变量名和成对的参数（缓存变量名 + 标志值）
- 遍历参数对，逐一调用 `add_check_cxx_compiler_flag_impl` 进行检测
- 通过 `PARENT_SCOPE` 将结果传回调用者作用域

### 宏：`delayed_install`

延迟安装机制的注册宏：

- 参数：`base`（基础路径）、`files`（文件列表）、`destination`（目标路径）
- 将文件和目标路径记录到全局属性中，而非立即执行安装
- 支持绝对路径和相对路径（相对路径会与 base 拼接）
- 用途：避免安装过程中目录被清空导致文件丢失（主要用于 Cycles 插件安装）

### 函数：`delayed_do_install`

执行所有延迟安装操作：

- 使用 `function` 而非 `macro` 定义，确保 `${BUILD_TYPE}` 等变量不会在调用处被提前展开
- 从全局属性中读取文件列表和目标路径
- 逐一调用 `install(FILES ...)` 执行实际安装

### 宏：`remove_cc_flag`

从所有构建类型的 C 和 C++ 编译器标志中移除指定标志：

- 使用正则表达式替换，同时作用于 C 和 C++ 的所有构建类型变量
- 覆盖 Debug、Release、MinSizeRel、RelWithDebInfo 四种构建配置

### 宏：`remove_extra_strict_flags`

移除额外的严格编译警告标志：

- GCC 编译器：移除 `-Wunused-parameter`
- Clang 编译器：移除 `-Wunused-parameter`
- MSVC：预留了 TODO，尚未实现

## 依赖关系

- 使用 CMake 内置模块 `CheckCXXCompilerFlag`
- 无外部 cmake 文件依赖
