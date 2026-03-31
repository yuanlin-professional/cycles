# FindSDL2.cmake - SDL2 查找模块

## 概述

该模块用于查找 SDL2（Simple DirectMedia Layer 2）库。SDL2 是一个跨平台的多媒体开发库，提供对音频、键盘、鼠标、游戏手柄和图形硬件的低级别访问。Cycles 独立版渲染器使用 SDL2 创建窗口和处理用户输入事件。

## 查找逻辑

### 搜索路径

模块按以下优先级搜索 SDL2：

1. CMake 变量 `SDL2_ROOT_DIR`（若已定义）
2. 环境变量 `SDL2_ROOT_DIR`

### 搜索过程

1. **头文件搜索**：在搜索路径的以下子目录中查找 `SDL.h`：
   - `include/SDL2`
   - `include`
   - `SDL2`
2. **库文件搜索**：在 `lib64` 和 `lib` 子目录中查找名为 `SDL2` 的库文件。

### 版本要求

模块未指定最低版本要求。

## 输出变量

| 变量名 | 说明 |
|--------|------|
| `SDL2_FOUND` | 布尔值，指示是否成功找到 SDL2 |
| `SDL2_INCLUDE_DIRS` | SDL2 头文件目录列表 |
| `SDL2_LIBRARIES` | 需要链接的 SDL2 库列表 |
| `SDL2_ROOT_DIR` | SDL2 的搜索基础目录（支持环境变量） |

### 内部变量（非公开接口）

| 变量名 | 说明 |
|--------|------|
| `SDL2_LIBRARY` | SDL2 库文件路径 |

## 依赖关系

- 无直接 CMake 模块依赖
- 使用 CMake 内置的 `FindPackageHandleStandardArgs` 模块
