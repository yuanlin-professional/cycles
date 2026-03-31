# FindOpenVDB.cmake - OpenVDB 查找模块

## 概述

该模块用于查找 OpenVDB 库。OpenVDB 是由 DreamWorks Animation 开发的开源体积数据结构库，专门用于高效存储和操作稀疏体积数据（如烟雾、火焰、水等效果）。Cycles 渲染器使用 OpenVDB 加载和渲染体积数据。

## 查找逻辑

### 搜索路径

模块按以下优先级搜索 OpenVDB：

1. CMake 变量 `OPENVDB_ROOT_DIR`（若已定义）
2. 环境变量 `OPENVDB_ROOT_DIR`
3. 系统路径 `/opt/lib/openvdb`

### 搜索过程

1. **头文件搜索**：在搜索路径的 `include` 子目录中查找 `openvdb/openvdb.h`。
2. **库文件搜索**：在 `lib64` 和 `lib` 子目录中查找名为 `openvdb` 的库文件。

### 版本要求

模块未指定最低版本要求。

## 输出变量

| 变量名 | 说明 |
|--------|------|
| `OPENVDB_FOUND` | 布尔值，指示是否成功找到 OpenVDB |
| `OPENVDB_INCLUDE_DIRS` | OpenVDB 头文件目录列表 |
| `OPENVDB_LIBRARIES` | 需要链接的 OpenVDB 库列表 |
| `OPENVDB_ROOT_DIR` | OpenVDB 的搜索基础目录（支持环境变量） |

### 内部变量（非公开接口）

| 变量名 | 说明 |
|--------|------|
| `OPENVDB_LIBRARY` | OpenVDB 库文件路径 |

## 依赖关系

- 无直接 CMake 模块依赖
- 使用 CMake 内置的 `FindPackageHandleStandardArgs` 模块
- OpenVDB 可能被 `FindUSDHoudini.cmake` 和 `FindUSDPixar.cmake` 模块覆盖设置
- OpenVDB 运行时依赖 TBB（线程构建模块）
