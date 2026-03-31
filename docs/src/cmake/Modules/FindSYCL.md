# FindSYCL.cmake - SYCL 查找模块

## 概述

该模块用于查找 SYCL 库和编译器。SYCL 是由 Khronos Group 制定的基于 C++ 的异构计算编程模型，允许在多种硬件（CPU、GPU、FPGA 等）上运行并行代码。此模块专门针对 Intel oneAPI 的 DPC++ 实现进行查找。Cycles 渲染器使用 SYCL/oneAPI 作为 Intel GPU 的渲染后端。

## 查找逻辑

### 搜索路径

模块按以下优先级搜索 SYCL：

1. CMake 变量 `SYCL_ROOT_DIR`（若已定义）
2. 环境变量 `SYCL_ROOT_DIR`
3. 系统路径 `/usr/lib`
4. 系统路径 `/usr/local/lib`
5. Linux 上的 Intel oneAPI 默认路径：`/opt/intel/oneapi/compiler/latest/linux/`
6. Windows 上的 Intel oneAPI 默认路径：`C:/Program Files (x86)/Intel/oneAPI/compiler/latest/windows`

### 搜索过程

1. **编译器搜索**（两阶段）：
   - 第一阶段：在指定搜索路径的 `bin` 子目录中查找 `icpx`、`dpcpp` 或 `clang++`（不搜索系统默认路径以避免误匹配系统 CLang）
   - 第二阶段（回退）：仅搜索 `icpx` 和 `dpcpp`，不包含 `clang++`（避免将系统级 CLang 编译器误认为 SYCL 编译器）
2. **库文件搜索**：在 `lib64` 和 `lib` 子目录中依次查找 `sycl10`、`sycl9`、`sycl8`、`sycl7`、`sycl6`、`sycl`。
3. **Windows 调试库搜索**：在 Windows 平台上额外查找带 `d` 后缀的调试版库（如 `sycl10d`、`sycl9d` 等）。
4. **头文件搜索**：在 `include` 子目录中查找 `sycl/sycl.hpp`。
5. **版本检测**：若找到 `sycl/version.hpp`，则从中解析 `__LIBSYCL_MAJOR_VERSION`、`__LIBSYCL_MINOR_VERSION` 和 `__LIBSYCL_PATCH_VERSION` 宏定义以获取版本号。

### 版本要求

模块会自动检测 SYCL 版本并存储在 `SYCL_VERSION` 变量中。

### 平台特殊处理

- **Windows**：当调试库可用时，`SYCL_LIBRARIES` 使用 `optimized`/`debug` 关键字区分发布和调试版本
- **其他平台**：`SYCL_LIBRARIES` 仅包含发布版库

## 输出变量

| 变量名 | 说明 |
|--------|------|
| `SYCL_FOUND` | 布尔值，指示是否成功找到 SYCL |
| `SYCL_COMPILER` | SYCL/DPC++ 编译器的完整路径 |
| `SYCL_INCLUDE_DIR` | SYCL 头文件目录（包含 `sycl/` 子目录） |
| `SYCL_LIBRARIES` | 需要链接的 SYCL 库列表 |
| `SYCL_VERSION` | 检测到的 SYCL 版本号（格式：`主版本.次版本.补丁`） |
| `SYCL_ROOT_DIR` | SYCL 的搜索基础目录（支持环境变量） |

### 内部变量（非公开接口）

| 变量名 | 说明 |
|--------|------|
| `SYCL_LIBRARY` | SYCL 发布版库文件路径 |
| `SYCL_LIBRARY_DEBUG` | SYCL 调试版库文件路径（仅 Windows） |

## 依赖关系

- 无直接 CMake 模块依赖
- 使用 CMake 内置的 `FindPackageHandleStandardArgs` 模块
- 运行时依赖 Intel oneAPI 运行时环境和兼容的 Intel GPU 驱动
