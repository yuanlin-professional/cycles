# FindPythonLibsUnix.cmake - Unix 平台 Python 库查找模块

## 概述

该模块用于在 Unix/Linux/macOS 平台上查找 Python 库。此模块是专门为 Blender/Cycles 项目定制的 Python 查找模块，硬编码了特定的 Python 版本（当前为 3.11），因为 Blender 在同一时期仅支持单一 Python 版本。该模块不适用于 Windows 平台，也不作为通用 Python 查找模块供其他项目使用。

## 查找逻辑

### 搜索路径

模块按以下优先级搜索 Python：

1. CMake 变量 `PYTHON_ROOT_DIR`（若已定义）
2. 环境变量 `PYTHON_ROOT_DIR`
3. 用户主目录 `$HOME/py${PYTHON_VERSION_NO_DOTS}`
4. 系统路径 `/opt/lib/python-${PYTHON_VERSION}`

### 搜索过程

1. **ABI 标志检测**：遍历测试 ABI 标志组合（`u` 为发布版，`du`/`d` 为调试版），确保从同一 ABI 版本中找到所有组件。
2. **头文件搜索**：在 `include/python${VERSION}${ABI_FLAGS}` 和 `include/${CMAKE_LIBRARY_ARCHITECTURE}/python${VERSION}${ABI_FLAGS}` 路径下查找 `Python.h`。
3. **配置头文件搜索**：查找 `pyconfig.h`，若未找到则回退到与 `Python.h` 相同的目录。
4. **库文件搜索**：在 `lib64` 和 `lib` 子目录中查找 `python${VERSION}${ABI_FLAGS}` 库文件。
5. **库路径搜索**：通过查找 `python${VERSION}/abc.py` 文件定位 Python 库路径。若失败则回退到库文件所在目录。
6. **site-packages 搜索**：在库路径中查找 `dist-packages`（Debian 特有）或 `site-packages` 目录。
7. **Python 可执行文件搜索**：在 `bin` 子目录中查找 Python 解释器。

### 版本要求

- 官方支持版本为 Python 3.11
- 用户可通过 `PYTHON_VERSION` 缓存变量覆盖版本号
- 使用非官方版本时会显示不同的警告消息

### 平台特殊处理

- **macOS**：当构建 Python 模块（`WITH_PYTHON_MODULE`）时，使用 `-undefined dynamic_lookup` 链接标志
- **其他 Unix**：使用 `-Xlinker -export-dynamic` 链接标志（可通过 `PYTHON_LINKFLAGS` 缓存变量自定义）

## 输出变量

| 变量名 | 说明 |
|--------|------|
| `PYTHONLIBSUNIX_FOUND` | 布尔值，指示是否成功找到 Python |
| `PYTHON_VERSION` | Python 版本号（主版本号.次版本号，如 `3.11`） |
| `PYTHON_VERSION_NO_DOTS` | 无点号的版本号（如 `311`） |
| `PYTHON_INCLUDE_DIRS` | Python 头文件目录列表（包含 `Python.h` 和 `pyconfig.h` 目录） |
| `PYTHON_INCLUDE_CONFIG_DIRS` | Python 配置头文件目录 |
| `PYTHON_LIBRARIES` | 需要链接的 Python 库列表（构建 Python 模块时不设置） |
| `PYTHON_LIBPATH` | Python 库安装路径（用于部署安装） |
| `PYTHON_SITE_PACKAGES` | Python site-packages 目录路径（用于模块安装） |
| `PYTHON_LINKFLAGS` | Python 链接标志 |
| `PYTHON_EXECUTABLE` | Python 解释器可执行文件路径 |
| `PYTHON_ROOT_DIR` | Python 的搜索基础目录（支持环境变量） |

### 内部变量（非公开接口）

| 变量名 | 说明 |
|--------|------|
| `PYTHON_LIBRARY` | Python 库文件路径 |

## 依赖关系

- 无直接 CMake 模块依赖
- 使用 CMake 内置的 `FindPackageHandleStandardArgs` 模块
- 仅适用于 Unix 系列平台（Linux、macOS 等），不适用于 Windows
- 受 `WITH_PYTHON_MODULE` 选项影响链接行为
