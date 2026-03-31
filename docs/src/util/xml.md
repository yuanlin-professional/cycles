# xml.h - PugiXML 封装与命名空间映射

## 概述

`xml.h` 为 Cycles 提供对 PugiXML 库的封装，将 PugiXML 的核心类型引入 Cycles 命名空间。支持两种 PugiXML 来源：系统安装的独立版本和 OpenImageIO (OIIO) 内嵌版本。仅在编译时定义 `WITH_PUGIXML` 时生效。

## 类与结构体

无自定义类或结构体。本文件仅通过 `using` 声明引入 PugiXML 类型。

## 核心函数/宏定义

### 条件编译宏

| 宏 | 条件 | 说明 |
|----|------|------|
| `PUGIXML_NAMESPACE` | `WITH_SYSTEM_PUGIXML` 定义时 | 映射为 `pugi`（系统版本） |
| `PUGIXML_NAMESPACE` | `WITH_SYSTEM_PUGIXML` 未定义时 | 映射为 `OIIO_NAMESPACE::pugi`（OIIO 内嵌版本） |

### 引入的类型

| 类型 | 说明 |
|------|------|
| `xml_attribute` | XML 属性 |
| `xml_document` | XML 文档对象 |
| `xml_node` | XML 节点 |
| `xml_parse_result` | XML 解析结果 |

## 依赖关系

- **内部头文件**: 无
- **外部依赖**: `<pugixml.hpp>`（PugiXML 库）
- **被引用**: `app/cycles_xml.cpp`

## 实现细节

1. **双来源支持**：Cycles 可以使用系统安装的 PugiXML（`WITH_SYSTEM_PUGIXML`）或 OIIO 捆绑的版本。两者的命名空间不同，通过 `PUGIXML_NAMESPACE` 宏统一处理。

2. **条件启用**：整个文件内容被 `#ifdef WITH_PUGIXML` 包裹。不需要 XML 功能的构建配置不会引入任何 PugiXML 依赖。

3. **用途**：主要用于 Cycles 独立应用的 XML 场景文件解析（`cycles_xml.cpp`），Blender 集成模式下通常不使用此路径。

## 关联文件

- `app/cycles_xml.cpp` - XML 场景文件加载器，使用本文件提供的类型
