# FindUSDHoudini.cmake - Houdini 环境中的 USD 查找模块

## 概述

该模块用于在 SideFX Houdini 安装目录中查找 USD（Universal Scene Description）库及相关依赖。与通用的 `FindUSD.cmake` 不同，此模块专门从 Houdini 的集成环境中提取 USD、OpenSubdiv、OpenVDB、TBB 和 OpenColorIO 等库，确保所有依赖版本与 Houdini 内置版本一致。输出变量格式与 `FindUSDPixar.cmake` 兼容。

## 查找逻辑

### 搜索路径

模块依赖 `HOUDINI_ROOT` 变量指向 Houdini 安装根目录。

### 平台特定路径

| 平台 | 库目录 | 头文件目录 |
|------|--------|------------|
| Windows | `${HOUDINI_ROOT}/custom/houdini/dsolib` | `${HOUDINI_ROOT}/toolkit/include` |
| macOS | `${HOUDINI_ROOT}/Frameworks/Houdini.framework/Libraries` | `${HOUDINI_ROOT}/Frameworks/Houdini.framework/Resources/toolkit/include` |
| Linux | `${HOUDINI_ROOT}/dsolib` | `${HOUDINI_ROOT}/toolkit/include` |

### 搜索过程

1. **Houdini 检测**：检查 `HOUDINI_ROOT` 是否已定义且路径存在。
2. **版本检测**：从 `HAPI/HAPI_Version.h` 中解析 Houdini 主版本号。
3. **USD 库搜索**：查找以 `pxr_` 为前缀的 Houdini 版 USD 库集合，包括：`hd`、`hgi`、`hgiGL`、`gf`、`arch`、`garch`、`plug`、`tf`、`trace`、`vt`、`work`、`sdf`、`cameraUtil`、`hf`、`pxOsd`、`usd`、`usdImaging`、`usdGeom`。
4. **Python 搜索**：自动检测 Houdini 内置的 Python 版本，查找 Python 库和 Boost.Python（hboost）库。
5. **依赖库搜索**：从 Houdini 安装目录中查找以下依赖：
   - OpenSubdiv（`osdCPU`、`osdGPU` 或 `osdCPU_md`、`osdGPU_md`）
   - OpenVDB（`openvdb_sesi`，Houdini 专用版本）
   - TBB（`tbb`）
   - OpenColorIO（`OpenColorIO_sidefx`，Houdini 专用版本）

### 版本要求

自动从 Houdini 安装中获取版本信息，无需手动指定。

## 输出变量

| 变量名 | 说明 |
|--------|------|
| `USD_FOUND` | 布尔值，指示是否成功找到 Houdini 中的 USD |
| `USD_INCLUDE_DIR` | USD 头文件目录 |
| `USD_INCLUDE_DIRS` | USD 头文件目录列表（包含 Python 头文件路径） |
| `USD_LIBRARIES` | 需要链接的所有 USD 相关库列表 |
| `HOUDINI_VERSION_MAJOR` | 检测到的 Houdini 主版本号 |
| `OPENSUBDIV_INCLUDE_DIRS` | Houdini 内置 OpenSubdiv 头文件目录 |
| `OPENSUBDIV_LIBRARIES` | Houdini 内置 OpenSubdiv 库列表 |
| `USD_OVERRIDE_OPENSUBDIV` | 设为 `ON`，指示使用 Houdini 的 OpenSubdiv |
| `OPENVDB_INCLUDE_DIRS` | Houdini 内置 OpenVDB 头文件目录 |
| `OPENVDB_LIBRARIES` | Houdini 内置 OpenVDB 库列表 |
| `USD_OVERRIDE_OPENVDB` | 设为 `ON`，指示使用 Houdini 的 OpenVDB |
| `TBB_INCLUDE_DIRS` | Houdini 内置 TBB 头文件目录 |
| `TBB_LIBRARIES` | Houdini 内置 TBB 库列表 |
| `USD_OVERRIDE_TBB` | 设为 `ON`，指示使用 Houdini 的 TBB |
| `OPENCOLORIO_INCLUDE_DIRS` | Houdini 内置 OpenColorIO 头文件目录 |
| `OPENCOLORIO_LIBRARIES` | Houdini 内置 OpenColorIO 库列表 |
| `USD_OVERRIDE_OPENCOLORIO` | 设为 `ON`，指示使用 Houdini 的 OpenColorIO |
| `BOOST_DEFINITIONS` | Boost 编译定义（设为 `-DHBOOST_ALL_NO_LIB`） |

## 依赖关系

- 需要有效的 Houdini 安装目录（通过 `HOUDINI_ROOT` 指定）
- 覆盖以下独立查找模块的输出：`FindOpenSubdiv.cmake`、`FindOpenVDB.cmake`、`FindTBB.cmake`
- 额外覆盖 OpenColorIO 的设置
- 输出变量与 `FindUSDPixar.cmake` 兼容，二者可互换使用
- Windows 平台上会向 `CMAKE_FIND_LIBRARY_PREFIXES` 添加 `lib` 和空字符串前缀
