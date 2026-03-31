# FindOSL.cmake - OpenShadingLanguage 查找模块

## 概述

该模块用于查找 OSL（Open Shading Language）库。OSL 是由 Sony Pictures Imageworks 开发的着色语言，专为高级渲染器中的可编程着色设计。在 Cycles 渲染器中，OSL 是核心的着色系统之一，允许用户编写自定义着色器节点，实现灵活的材质和纹理效果。

## 查找逻辑

模块按照以下优先级搜索 OSL：

1. **用户指定路径**：
   - 若定义了 `OSL_ROOT_DIR`（CMake 变量或环境变量），则优先在该路径下搜索。
   - 若定义了 `OSLHOME` 环境变量，则作为着色器文件的附加搜索路径。
2. **系统默认路径**：`/opt/lib/osl`

### 头文件搜索

在搜索目录的 `include` 子目录下查找 `OSL/oslversion.h`。

### 库文件搜索

模块搜索以下 OSL 组件库，在 `lib64`、`lib` 子目录下查找：

- `oslcomp`：OSL 编译器库
- `oslexec`：OSL 执行引擎库
- `oslquery`：OSL 查询库
- `oslnoise`：OSL 噪声库（可选，取决于版本）

链接顺序有特殊要求。在 macOS 上，`oslexec` 使用 `-force_load` 标志强制加载。

### 编译器和着色器搜索

- **OSL 编译器**：在搜索目录的 `bin` 子目录下查找 `oslc` 可执行文件。
- **着色器文件目录**：在多个路径下查找 `stdosl.h` 标准着色器头文件：
  - `${OSL_ROOT_DIR}/share/OSL/shaders`
  - `${OSL_COMPILER}/../shaders`
  - `${OSLHOME}/shaders`
  - `/usr/share/OSL/`、`/usr/include/OSL/`

### 版本检测

从 `OSL/oslversion.h` 头文件中提取以下版本信息：

- `OSL_LIBRARY_VERSION_MAJOR`：主版本号
- `OSL_LIBRARY_VERSION_MINOR`：次版本号
- `OSL_LIBRARY_VERSION_PATCH`：补丁版本号

## 输出变量

| 变量名 | 说明 |
|--------|------|
| `OSL_FOUND` | 布尔值，若为 `FALSE` 则表示未找到 OSL |
| `OSL_INCLUDE_DIRS` | OSL 头文件目录 |
| `OSL_LIBRARIES` | 链接 OSL 所需的全部组件库文件列表 |
| `OSL_COMPILER` | OSL 脚本编译器 `oslc` 的完整路径 |
| `OSL_SHADER_DIR` | OSL 标准着色器文件所在目录 |
| `OSL_ROOT_DIR` | 搜索 OSL 的根目录，可通过环境变量设置 |
| `OSL_LIBRARY_VERSION_MAJOR` | OSL 主版本号 |
| `OSL_LIBRARY_VERSION_MINOR` | OSL 次版本号 |
| `OSL_LIBRARY_VERSION_PATCH` | OSL 补丁版本号 |
| `OSL_VERSION` | OSL 完整版本字符串（主.次.补丁） |

### 内部变量（高级）

| 变量名 | 说明 |
|--------|------|
| `OSL_INCLUDE_DIR` | OSL 头文件目录（单数形式） |
| `OSL_SHADER_DIR` | OSL 着色器目录 |
| `OSL_OSLCOMP_LIBRARY` | oslcomp 库路径 |
| `OSL_OSLEXEC_LIBRARY` | oslexec 库路径 |
| `OSL_OSLQUERY_LIBRARY` | oslquery 库路径 |
| `OSL_OSLNOISE_LIBRARY` | oslnoise 库路径（可选） |

## 依赖关系

该模块使用 CMake 内置的 `FindPackageHandleStandardArgs` 模块。OSL 本身在运行时依赖 LLVM 进行 JIT 编译。

> **相关模块**：`FindLLVM.cmake` 和 `FindClang.cmake` 通常与本模块一起使用，因为 OSL 的编译和执行依赖 LLVM/Clang 基础设施。
