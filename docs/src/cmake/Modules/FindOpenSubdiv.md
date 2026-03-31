# FindOpenSubdiv.cmake - OpenSubdiv 查找模块

## 概述

该模块用于查找 OpenSubdiv 库。OpenSubdiv 是由 Pixar 开发的开源细分曲面库，提供高性能的细分曲面评估功能，支持 CPU 和 GPU 两种计算后端。Cycles 渲染器使用 OpenSubdiv 实现网格的细分曲面处理。

## 查找逻辑

### 搜索路径

模块按以下优先级搜索 OpenSubdiv：

1. CMake 变量 `OPENSUBDIV_ROOT_DIR`（若已定义）
2. 环境变量 `OPENSUBDIV_ROOT_DIR`
3. 系统路径 `/opt/lib/opensubdiv`
4. 系统路径 `/opt/lib/osd`（由 `install_deps.sh` 脚本安装时使用）

### 搜索过程

1. **头文件搜索**：在搜索路径的 `include` 子目录中查找 `opensubdiv/osd/mesh.h`。
2. **组件库搜索**：遍历以下两个组件，在 `lib64` 和 `lib` 子目录中分别查找对应的库文件：
   - `osdGPU` — GPU 计算后端库
   - `osdCPU` — CPU 计算后端库
3. **控制器检测**：提供 `OPENSUBDIV_CHECK_CONTROLLER` 宏，用于检测特定控制器头文件是否存在，以判断 OpenSubdiv 是否支持特定功能。

### 版本要求

模块未指定最低版本要求。

## 输出变量

| 变量名 | 说明 |
|--------|------|
| `OPENSUBDIV_FOUND` | 布尔值，指示是否成功找到 OpenSubdiv |
| `OPENSUBDIV_INCLUDE_DIRS` | OpenSubdiv 头文件目录列表 |
| `OPENSUBDIV_LIBRARIES` | 需要链接的 OpenSubdiv 库列表（包含 osdGPU 和 osdCPU） |
| `OPENSUBDIV_ROOT_DIR` | OpenSubdiv 的搜索基础目录（支持环境变量） |

### 内部变量（非公开接口）

| 变量名 | 说明 |
|--------|------|
| `OPENSUBDIV_OSDGPU_LIBRARY` | osdGPU 库文件路径 |
| `OPENSUBDIV_OSDCPU_LIBRARY` | osdCPU 库文件路径 |

## 依赖关系

- 无直接 CMake 模块依赖
- 使用 CMake 内置的 `FindPackageHandleStandardArgs` 模块
- OpenSubdiv 可能被 `FindUSDHoudini.cmake` 和 `FindUSDPixar.cmake` 模块覆盖设置
