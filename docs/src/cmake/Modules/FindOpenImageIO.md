# FindOpenImageIO.cmake - OpenImageIO 查找模块

## 概述

该模块用于查找 OpenImageIO（OIIO）库。OpenImageIO 是一个用于读取、写入和处理多种图像文件格式的开源库，广泛应用于视觉特效和动画制作流程中。Cycles 渲染器使用 OIIO 进行纹理加载和图像输入/输出操作。

## 查找逻辑

### 搜索路径

模块按以下优先级搜索 OpenImageIO：

1. CMake 变量 `OPENIMAGEIO_ROOT_DIR`（若已定义）
2. 环境变量 `OPENIMAGEIO_ROOT_DIR`
3. 系统路径 `/opt/lib/oiio`

### 搜索过程

1. **头文件搜索**：在搜索路径的 `include` 子目录中查找 `OpenImageIO/imageio.h`。
2. **主库搜索**：在 `lib64` 和 `lib` 子目录中查找 `OpenImageIO` 库文件。
3. **工具搜索**：在 `bin` 子目录中查找 `oiiotool` 命令行工具。
4. **Util 库检测**：检查 `export.h` 中是否定义了 `OIIO_UTIL_API` 宏。若存在，说明新版本 OIIO 将工具库拆分为独立的 `OpenImageIO_Util` 库，需额外链接。旧版本中该库包含在主库中，同时链接两者会导致符号重复。
5. **内置 PugiXML 检测**：检查 `OpenImageIO/pugixml.hpp` 是否存在，以判断 OIIO 是否内置了 PugiXML 解析器。

### 版本要求

模块未指定最低版本要求，但会自动适配新旧版本的库拆分差异。

## 输出变量

| 变量名 | 说明 |
|--------|------|
| `OPENIMAGEIO_FOUND` | 布尔值，指示是否成功找到 OpenImageIO |
| `OPENIMAGEIO_INCLUDE_DIRS` | OpenImageIO 头文件目录列表 |
| `OPENIMAGEIO_LIBRARIES` | 需要链接的 OpenImageIO 库列表（可能包含主库和 Util 库） |
| `OPENIMAGEIO_PUGIXML_FOUND` | 布尔值，指示 OIIO 是否内置 PugiXML 解析器 |
| `OPENIMAGEIO_TOOL` | `oiiotool` 可执行文件的完整路径 |
| `OPENIMAGEIO_ROOT_DIR` | OpenImageIO 的搜索基础目录（支持环境变量） |

### 内部变量（非公开接口）

| 变量名 | 说明 |
|--------|------|
| `OPENIMAGEIO_LIBRARY` | OpenImageIO 主库文件路径 |
| `OPENIMAGEIO_UTIL_LIBRARY` | OpenImageIO_Util 库文件路径（仅新版本存在） |

## 依赖关系

- 无直接 CMake 模块依赖
- 使用 CMake 内置的 `FindPackageHandleStandardArgs` 模块
- OpenImageIO 本身可能依赖 PugiXML（内置或外部）
- 与 `FindPugiXML.cmake` 模块存在关联：当 OIIO 未内置 PugiXML 时，可能需要单独查找
