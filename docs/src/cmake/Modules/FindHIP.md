# FindHIP.cmake - HIP 查找模块

## 概述

该模块用于查找 AMD HIP（Heterogeneous-compute Interface for Portability）编译器工具链。HIP 是 AMD 推出的 GPU 编程平台，允许开发者编写可在 AMD 和 NVIDIA GPU 上运行的可移植代码。在 Cycles 渲染器中，HIP 用于支持 AMD GPU 的硬件加速光线追踪渲染。

## 查找逻辑

模块按照以下优先级搜索 HIP：

1. **用户指定路径**：若定义了 `HIP_ROOT_DIR`（CMake 变量），则优先在该路径下搜索。
2. **环境变量**：
   - `HIP_ROOT_DIR` 环境变量
   - `HIP_PATH` 环境变量（AMD SDK 内置环境变量）
3. **系统默认路径**：
   - `/opt/rocm`
   - `/opt/rocm/hip`
   - `C:/Program Files/AMD/ROCm/*`（Windows）

### 编译器搜索

在搜索目录的 `bin` 子目录下查找 `hipcc` 可执行文件。

### 版本检测

若找到 `hipcc`，模块会从 `${HIP_ROOT_DIR}/include/hip/hip_version.h` 头文件中提取版本信息：

- `HIP_VERSION_MAJOR`：主版本号
- `HIP_VERSION_MINOR`：次版本号
- `HIP_VERSION_PATCH`：补丁版本号

并构建完整的语义版本字符串 `HIP_VERSION`（如 `5.7.1`）和短版本字符串 `HIP_VERSION_SHORT`（如 `5.7`）。

## 输出变量

| 变量名 | 说明 |
|--------|------|
| `HIP_FOUND` | 布尔值，若为 `FALSE` 则表示未找到 HIP 工具链 |
| `HIP_HIPCC_EXECUTABLE` | `hipcc` 编译器的完整路径 |
| `HIP_VERSION` | HIP 完整版本字符串（主.次.补丁） |
| `HIP_VERSION_SHORT` | HIP 短版本字符串（主.次） |
| `HIP_VERSION_MAJOR` | HIP 主版本号 |
| `HIP_VERSION_MINOR` | HIP 次版本号 |
| `HIP_VERSION_PATCH` | HIP 补丁版本号 |
| `HIP_ROOT_DIR` | HIP 安装根目录 |

## 依赖关系

该模块使用 CMake 内置的 `FindPackageHandleStandardArgs` 模块来处理标准的查找参数。无其他外部 Find 模块依赖。

> **注意**：该模块仅查找 HIP 编译器工具链，不查找 HIP 运行时库。HIP 运行时追踪相关功能由 `FindHIPRT.cmake` 模块提供。
