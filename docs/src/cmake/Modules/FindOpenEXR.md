# FindOpenEXR.cmake - OpenEXR 查找模块

## 概述

该模块用于查找 OpenEXR 库。OpenEXR 是由 Industrial Light & Magic 开发的高动态范围（HDR）图像文件格式库，广泛用于电影和视觉特效行业。在 Cycles 渲染器中，OpenEXR 用于读写 HDR 图像文件，包括渲染输出、环境贴图和多层 EXR 合成数据。

## 查找逻辑

模块按照以下优先级搜索 OpenEXR：

1. **用户指定路径**：若定义了 `OPENEXR_ROOT_DIR`（CMake 变量或环境变量），则优先在该路径下搜索。
2. **系统默认路径**：`/opt/lib/openexr`

### 头文件搜索

在搜索目录的 `include` 子目录下查找 `OpenEXR/ImfXdr.h`。

### 版本检测

从 `OpenEXRConfig.h`（位于 `${OPENEXR_INCLUDE_DIR}` 或 `${OPENEXR_INCLUDE_DIR}/OpenEXR`）中提取 `OPENEXR_VERSION_STRING` 宏定义。若无法找到版本信息，默认假设版本为 `2.0`。

### 库文件搜索

根据 OpenEXR 版本不同，搜索的组件库不同：

**OpenEXR >= 3.0.0**：
- `Iex`
- `OpenEXR`
- `OpenEXRCore`
- `IlmThread`
- 另外还需查找独立的 **Imath** 库

**OpenEXR < 3.0.0**：
- `Half`
- `Iex`
- `IlmImf`
- `IlmThread`
- `Imath`

库文件搜索支持带版本后缀（如 `Iex-3_1`）和不带后缀两种命名方式，在 `lib64`、`lib` 子目录下查找。

### Imath 独立库处理（OpenEXR >= 3.0.0）

OpenEXR 3.0 开始，Imath 库被拆分为独立项目。模块会额外查找：

- **Imath 头文件**：查找 `Imath/ImathMath.h`
- **Imath 版本**：从 `ImathConfig.h` 中提取 `IMATH_VERSION_STRING`
- **Imath 库文件**：查找带或不带版本后缀的 `Imath` 库

## 输出变量

| 变量名 | 说明 |
|--------|------|
| `OPENEXR_FOUND` | 布尔值，若为 `FALSE` 则表示未找到 OpenEXR |
| `OPENEXR_INCLUDE_DIRS` | OpenEXR 头文件目录列表（包含 `OpenEXR/` 子目录，以及 3.0+ 的 `Imath/` 子目录） |
| `OPENEXR_LIBRARIES` | 链接 OpenEXR 所需的全部库文件列表（3.0+ 包含 Imath 库） |
| `OPENEXR_ROOT_DIR` | 搜索 OpenEXR 的根目录，可通过环境变量设置 |
| `OPENEXR_VERSION` | OpenEXR 版本字符串（缓存变量） |
| `IMATH_LIBRARIES` | Imath 库文件（3.0+ 为独立 Imath 库，旧版本为 OpenEXR 内置的 Imath 库） |
| `IMATH_INCLUDE_DIRS` | Imath 头文件目录 |

### 内部变量（高级）

| 变量名 | 说明 |
|--------|------|
| `OPENEXR_INCLUDE_DIR` | OpenEXR 头文件目录（单数形式） |
| `OPENEXR_HALF_LIBRARY` | Half 库路径（仅 < 3.0） |
| `OPENEXR_IEX_LIBRARY` | Iex 库路径 |
| `OPENEXR_ILMIMF_LIBRARY` | IlmImf 库路径（仅 < 3.0） |
| `OPENEXR_ILMTHREAD_LIBRARY` | IlmThread 库路径 |
| `OPENEXR_IMATH_LIBRARY` | Imath 库路径（仅 < 3.0） |
| `OPENEXR_OPENEXR_LIBRARY` | OpenEXR 库路径（仅 >= 3.0） |
| `OPENEXR_OPENEXRCORE_LIBRARY` | OpenEXRCore 库路径（仅 >= 3.0） |
| `IMATH_INCLUDE_DIR` | Imath 头文件目录（仅 >= 3.0） |
| `IMATH_LIBRARY` | Imath 库路径（仅 >= 3.0） |

## 依赖关系

该模块使用 CMake 内置的 `FindPackageHandleStandardArgs` 模块来处理标准的查找参数。

> **注意**：OpenEXR >= 3.0.0 版本将 Imath 拆分为独立项目。本模块会自动处理这一变化，将 Imath 的头文件和库文件一并纳入 OpenEXR 的输出变量中，以简化对 2.x 和 3.x 两个大版本的兼容支持。
