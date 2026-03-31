# FindTBB.cmake - TBB 查找模块

## 概述

该模块用于查找 TBB（Threading Building Blocks）库。TBB 是由 Intel 开发的 C++ 并行编程库（现已更名为 oneTBB），提供丰富的并行算法、并发容器和任务调度器。Cycles 渲染器使用 TBB 实现多线程并行渲染和其他并行计算任务。

## 查找逻辑

### 搜索路径

模块按以下优先级搜索 TBB：

1. CMake 变量 `TBB_ROOT_DIR`（若已定义）
2. 环境变量 `TBB_ROOT_DIR`
3. 系统路径 `/opt/lib/tbb`

### 搜索过程

1. **头文件搜索**：在搜索路径的 `include` 子目录中查找 `tbb/tbb.h`。
2. **库文件搜索**：在 `lib64` 和 `lib` 子目录中查找名为 `tbb` 的库文件。

### 版本要求

模块未指定最低版本要求。

## 输出变量

| 变量名 | 说明 |
|--------|------|
| `TBB_FOUND` | 布尔值，指示是否成功找到 TBB |
| `TBB_INCLUDE_DIRS` | TBB 头文件目录列表 |
| `TBB_LIBRARIES` | 需要链接的 TBB 库列表 |
| `TBB_ROOT_DIR` | TBB 的搜索基础目录（支持环境变量） |

### 内部变量（非公开接口）

| 变量名 | 说明 |
|--------|------|
| `TBB_LIBRARY` | TBB 库文件路径 |

## 依赖关系

- 无直接 CMake 模块依赖
- 使用 CMake 内置的 `FindPackageHandleStandardArgs` 模块
- TBB 可能被 `FindUSDHoudini.cmake` 和 `FindUSDPixar.cmake` 模块覆盖设置
- OpenVDB 等多个库在运行时依赖 TBB
