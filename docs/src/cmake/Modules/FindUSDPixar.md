# FindUSDPixar.cmake - Pixar USD 安装环境查找模块

## 概述

该模块用于在 Pixar USD 的标准安装目录中查找 Hydra、TBB、OpenSubdiv 和 OpenVDB 库。与通用的 `FindUSD.cmake` 不同，此模块通过 CMake 的 `find_package(pxr CONFIG)` 机制利用 Pixar USD 自带的 CMake 配置文件进行查找。输出变量格式与 `FindUSDHoudini.cmake` 兼容。

## 查找逻辑

### 搜索路径

模块依赖 `PXR_ROOT` 变量指向 Pixar USD 安装根目录。

### 搜索过程

1. **USD 配置查找**：调用 `find_package(pxr CONFIG REQUIRED PATHS ${PXR_ROOT} NO_DEFAULT_PATH)` 使用 Pixar USD 的 CMake 配置文件。
2. **USD 库设置**：设置需要链接的 USD 库列表：`hd`、`hgi`、`hgiGL`、`usd`、`usdImaging`、`usdGeom`。
3. **OpenSubdiv 搜索**：在 `${PXR_CMAKE_DIR}/lib` 中查找 OpenSubdiv 的 CPU 和 GPU 库，同时查找调试版（`osdCPU_d`、`osdGPU_d`）和发布版（`osdCPU`、`osdGPU`），使用 `optimized`/`debug` 关键字区分。
4. **OpenVDB 搜索**：在 USD 库目录中查找 `openvdb` 库。
5. **TBB 搜索**：在 USD 库目录中查找 `tbb` 和 `tbb_debug` 库，使用 `optimized`/`debug` 关键字区分。

### 版本要求

由 Pixar USD 的 CMake 配置文件自动处理版本信息。

## 输出变量

| 变量名 | 说明 |
|--------|------|
| `USD_FOUND` | 布尔值，指示是否成功找到 Pixar USD |
| `USD_INCLUDE_DIRS` | USD 头文件目录列表（来自 `PXR_INCLUDE_DIRS`） |
| `USD_LIBRARIES` | 需要链接的 USD 库列表 |
| `OPENSUBDIV_INCLUDE_DIRS` | USD 安装中的 OpenSubdiv 头文件目录（仅当找到时设置） |
| `OPENSUBDIV_LIBRARIES` | USD 安装中的 OpenSubdiv 库列表（包含 debug/optimized 变体） |
| `USD_OVERRIDE_OPENSUBDIV` | 设为 `ON`，指示使用 USD 安装中的 OpenSubdiv（仅当找到时） |
| `OPENVDB_INCLUDE_DIRS` | USD 安装中的 OpenVDB 头文件目录（仅当找到时设置） |
| `OPENVDB_LIBRARIES` | USD 安装中的 OpenVDB 库列表 |
| `USD_OVERRIDE_OPENVDB` | 设为 `ON`，指示使用 USD 安装中的 OpenVDB（仅当找到时） |
| `TBB_INCLUDE_DIRS` | USD 安装中的 TBB 头文件目录（仅当找到时设置） |
| `TBB_LIBRARIES` | USD 安装中的 TBB 库列表（包含 debug/optimized 变体） |
| `USD_OVERRIDE_TBB` | 设为 `ON`，指示使用 USD 安装中的 TBB（仅当找到时） |

## 依赖关系

- 需要 Pixar USD 安装及其 CMake 配置文件（通过 `PXR_ROOT` 指定）
- 使用 CMake 的 `find_package(pxr CONFIG)` 机制
- 条件性覆盖以下独立查找模块的输出：`FindOpenSubdiv.cmake`、`FindOpenVDB.cmake`、`FindTBB.cmake`
- 输出变量与 `FindUSDHoudini.cmake` 兼容，二者可互换使用
- 与 `FindUSDHoudini.cmake` 不同的是，此模块对每个依赖库进行条件检测——仅在 USD 安装目录中实际存在该库时才覆盖设置
