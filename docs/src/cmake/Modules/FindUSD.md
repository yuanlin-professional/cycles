# FindUSD.cmake - USD 查找模块

## 概述

该模块用于查找 USD（Universal Scene Description，通用场景描述）库。USD 是由 Pixar 开发的开源场景描述框架，用于在不同数字内容创作（DCC）工具之间交换和协作 3D 场景数据。Cycles 渲染器通过 USD 的 Hydra 渲染委托接口集成为渲染后端。

## 查找逻辑

### 搜索路径

模块按以下优先级搜索 USD：

1. CMake 变量 `USD_ROOT_DIR`（若已定义）
2. 环境变量 `USD_ROOT_DIR`
3. 系统路径 `/opt/lib/usd`

### 搜索过程

1. **头文件搜索**：在搜索路径的 `include` 子目录中查找 `pxr/usd/usd/api.h`。
2. **库文件搜索**：在 `lib64`、`lib` 和 `lib/static` 子目录中查找 USD 库。支持多种命名约定：
   - `usd_usd_m` / `usd_usd_ms` — USD 21.11 及以后版本的带前缀单体库（monolithic）
   - `usd_m` / `usd_ms` — 旧版本的单体库
   - `${PXR_LIB_PREFIX}usd` — 使用自定义前缀的库
3. **Python 支持检测**：检查 `pxr/base/tf/pyModule.h` 是否存在以判断 USD 是否编译了 Python 支持。

### 版本要求

模块未指定最低版本要求，但自动适配 USD 21.11 引入的库命名变更（从 `libusd_m.a` 改为 `libusd_usd_m.a`）。

## 输出变量

| 变量名 | 说明 |
|--------|------|
| `USD_FOUND` | 布尔值，指示是否成功找到 USD |
| `USD_INCLUDE_DIRS` | USD 头文件目录列表 |
| `USD_LIBRARIES` | 需要链接的 USD 库列表 |
| `USD_LIBRARY_DIR` | USD 库文件所在目录 |
| `USD_PYTHON_SUPPORT` | 布尔值，指示 USD 是否支持 Python 绑定（当 `pyModule.h` 存在时为 `ON`） |
| `USD_ROOT_DIR` | USD 的搜索基础目录（支持环境变量） |

### 内部变量（非公开接口）

| 变量名 | 说明 |
|--------|------|
| `USD_LIBRARY` | USD 库文件路径 |

## 依赖关系

- 无直接 CMake 模块依赖
- 使用 CMake 内置的 `FindPackageHandleStandardArgs` 模块
- `PXR_LIB_PREFIX` 变量可选，用于指定库名前缀
- 与 `FindUSDHoudini.cmake` 和 `FindUSDPixar.cmake` 互为替代方案，三者输出变量格式兼容
