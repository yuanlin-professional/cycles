# FindOptiX.cmake - OptiX 查找模块

## 概述

该模块用于查找 NVIDIA OptiX 光线追踪引擎。OptiX 是 NVIDIA 提供的高性能 GPU 光线追踪 SDK，利用 NVIDIA RTX 硬件加速实现实时光线追踪。Cycles 渲染器使用 OptiX 作为 GPU 渲染后端之一，以获得硬件加速的光线追踪性能。

注意：OptiX 是仅头文件（header-only）的 SDK，因此该模块只查找头文件目录，不查找库文件。

## 查找逻辑

### 搜索路径

模块按以下优先级搜索 OptiX：

1. CMake 变量 `OPTIX_ROOT_DIR`（若已定义）
2. 环境变量 `OPTIX_ROOT_DIR`

### 搜索过程

1. **头文件搜索**：在搜索路径的 `include` 子目录中查找 `optix.h`。
2. **版本检测**：若找到 `optix.h`，则从文件中解析 `OPTIX_VERSION` 宏定义，通过数学运算提取主版本号、次版本号和补丁版本号：
   - 主版本号 = `OPTIX_VERSION / 10000`
   - 次版本号 = `(OPTIX_VERSION % 10000) / 100`
   - 补丁版本号 = `OPTIX_VERSION % 100`

### 版本要求

模块会自动检测 OptiX 版本并存储在 `OPTIX_VERSION` 变量中，可与 `find_package` 的版本参数配合使用进行版本检查。

## 输出变量

| 变量名 | 说明 |
|--------|------|
| `OPTIX_FOUND` | 布尔值，指示是否成功找到 OptiX |
| `OPTIX_INCLUDE_DIRS` | OptiX 头文件目录列表 |
| `OPTIX_VERSION` | 检测到的 OptiX 版本号（格式：`主版本.次版本.补丁`） |
| `OPTIX_ROOT_DIR` | OptiX 的搜索基础目录（支持环境变量） |

## 依赖关系

- 无直接 CMake 模块依赖
- 使用 CMake 内置的 `FindPackageHandleStandardArgs` 模块
- OptiX 运行时依赖 NVIDIA CUDA 工具包和支持 RTX 的 NVIDIA GPU 驱动
