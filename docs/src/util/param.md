# param.h - 参数值与类型描述封装

## 概述
本文件封装了 OpenImageIO（OIIO）库的参数值列表（`ParamValue`）和类型描述（`TypeDesc`）系统，用于在 Cycles 渲染器的各种数据对象上存储自定义属性，这些属性随后可在着色器中使用。同时引入了 OIIO 的统一字符串（`ustring`）类型，用于高效的字符串比较和哈希。

## 类与结构体

### 从 OIIO 引入的类型
| 类型 | 说明 |
|------|------|
| `ParamValue` | 参数值容器，存储键值对形式的自定义属性 |
| `TypeDesc` | 类型描述符，描述数据的基础类型、聚合方式和语义 |
| `ustring` | 统一字符串，内部使用唯一化存储，支持 O(1) 比较 |
| `ustringhash` | `ustring` 的哈希值类型 |

### 预定义类型常量
| 常量 | 说明 |
|------|------|
| `TypeFloat`, `TypeInt`, `TypeString` | 标量类型 |
| `TypeColor`, `TypePoint`, `TypeVector`, `TypeNormal` | 3 分量语义类型 |
| `TypeFloat2`, `TypeFloat4`, `TypeMatrix` | 多分量/矩阵类型 |
| `TypeRGBA` | 自定义的 4 分量颜色类型（`FLOAT, VEC4, COLOR`） |
| `TypeFloatArray4` | 4 元素浮点数组（无语义） |

## 核心函数
本文件无自定义函数，仅提供类型别名和常量定义。

## 依赖关系
- **外部依赖**: `<OpenImageIO/paramlist.h>`, `<OpenImageIO/typedesc.h>`, `<OpenImageIO/ustring.h>`
- **被引用**: `scene/shader.h`, `scene/attribute.h`, `scene/image.h` 等场景管理模块

## 关联文件
- `util/string.h` - 字符串基础设施
- `scene/attribute.h` - 使用 `TypeDesc` 定义属性类型
