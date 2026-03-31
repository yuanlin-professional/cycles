# FindPugiXML.cmake - PugiXML 查找模块

## 概述

该模块用于查找 PugiXML 库。PugiXML 是一个轻量级的 C++ XML 处理库，提供 DOM 风格的接口来解析、修改和序列化 XML 文档。Cycles 渲染器使用 PugiXML 处理各种 XML 格式的配置和数据文件。

## 查找逻辑

### 搜索路径

模块按以下优先级搜索 PugiXML：

1. CMake 变量 `PUGIXML_ROOT_DIR`（若已定义）
2. 环境变量 `PUGIXML_ROOT_DIR`
3. 系统路径 `/opt/lib/oiio`（与 OpenImageIO 共享安装路径）

### 搜索过程

1. **头文件搜索**：在搜索路径的 `include` 子目录中查找 `pugixml.hpp`。
2. **库文件搜索**：在 `lib64` 和 `lib` 子目录中查找名为 `pugixml` 的库文件。

### 版本要求

模块未指定最低版本要求。

## 输出变量

| 变量名 | 说明 |
|--------|------|
| `PUGIXML_FOUND` | 布尔值，指示是否成功找到 PugiXML |
| `PUGIXML_INCLUDE_DIRS` | PugiXML 头文件目录列表 |
| `PUGIXML_LIBRARIES` | 需要链接的 PugiXML 库列表 |
| `PUGIXML_ROOT_DIR` | PugiXML 的搜索基础目录（支持环境变量） |

### 内部变量（非公开接口）

| 变量名 | 说明 |
|--------|------|
| `PUGIXML_LIBRARY` | PugiXML 库文件路径 |

## 依赖关系

- 无直接 CMake 模块依赖
- 使用 CMake 内置的 `FindPackageHandleStandardArgs` 模块
- 与 `FindOpenImageIO.cmake` 存在关联：OpenImageIO 可能内置 PugiXML，此时无需独立查找。搜索路径 `/opt/lib/oiio` 表明 PugiXML 可能随 OIIO 一同安装
