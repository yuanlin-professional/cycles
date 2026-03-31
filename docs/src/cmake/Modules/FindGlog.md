# FindGlog.cmake - Google glog 查找模块

## 概述

该模块用于查找 Google glog 日志库。glog 是 Google 开发的 C++ 日志库，提供应用级别的日志记录功能。该模块最初来源于 Ceres Solver 项目。在 Cycles 渲染器中，glog 可用于调试和运行时日志记录。

## 查找逻辑

模块执行手动组件搜索，搜索路径包括：

1. **用户指定路径**：若定义了 `GLOG_ROOT_DIR`（CMake 变量或环境变量），则优先在 `${GLOG_ROOT_DIR}/include` 和 `${GLOG_ROOT_DIR}/lib` 下搜索。
2. **系统路径**：
   - `/usr/local/include`、`/usr/local/homebrew/include`（macOS）
   - `/opt/local/var/macports/software`（macOS MacPorts）
   - `/opt/local/include`、`/usr/include`
   - `/opt/lib/glog/include`
3. **Windows 路径后缀**：`glog/include`、`glog/Include`、`Glog/include`、`Glog/Include`

### 头文件搜索

查找 `glog/logging.h` 头文件。支持通过 `GLOG_INCLUDE_DIR_HINTS` 指定额外搜索目录。

### 库文件搜索

查找名为 `glog` 的库文件。搜索路径包括系统库目录以及 Windows 下的路径后缀变体。支持通过 `GLOG_LIBRARY_DIR_HINTS` 指定额外搜索目录。

### MSVC 兼容性

在 MSVC 编译器下，模块会临时修改 `CMAKE_FIND_LIBRARY_PREFIXES` 以处理带 `lib` 前缀的库名称，搜索完成后恢复原值。

### 版本要求

glog 源码中未提供版本信息记录，因此该模块无法提取版本号。

## 输出变量

| 变量名 | 说明 |
|--------|------|
| `GLOG_FOUND` | 布尔值，若为 `TRUE` 则表示找到 glog |
| `GLOG_INCLUDE_DIRS` | glog 头文件目录 |
| `GLOG_LIBRARIES` | 链接 glog 所需的库文件列表 |

### 控制变量

| 变量名 | 说明 |
|--------|------|
| `GLOG_ROOT_DIR` | 搜索 glog 的根目录，可通过环境变量设置 |
| `GLOG_INCLUDE_DIR_HINTS` | 额外的头文件搜索目录列表 |
| `GLOG_LIBRARY_DIR_HINTS` | 额外的库文件搜索目录列表 |

### 内部变量（高级）

| 变量名 | 说明 |
|--------|------|
| `GLOG_INCLUDE_DIR` | glog 头文件目录（单数形式） |
| `GLOG_LIBRARY` | glog 库文件路径（单数形式） |

## 依赖关系

该模块使用 CMake 内置的 `FindPackageHandleStandardArgs` 模块来处理标准的查找参数。无其他外部 Find 模块依赖。
