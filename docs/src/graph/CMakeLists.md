# CMakeLists.txt - graph 模块构建配置

## 概述

本文件是 Cycles 渲染器 `graph` 模块的 CMake 构建配置文件。该模块负责编译场景图节点 (节点) 类型系统的核心源代码，包括节点基类、类型注册机制、枚举映射和 XML 序列化功能。最终构建产物为 `cycles_graph` 静态库。

## 构建选项 / 变量

### INC
- **值**: `..`（父目录）
- **说明**: 包含路径设置为上一级目录（即 `src/`），使得模块内代码可以用 `graph/node.h`、`util/array.h` 等相对于 `src/` 的路径形式来引用头文件。

### INC_SYS
- **值**: 空
- **说明**: 无额外的系统包含路径。

### SRC
源文件列表：
- `node.cpp` — 节点基类实现
- `node_type.cpp` — 节点类型系统和 socket 类型实现
- `node_xml.cpp` — XML 序列化/反序列化实现（受 `WITH_PUGIXML` 编译开关控制）

### SRC_HEADERS
头文件列表：
- `node.h` — 节点基类声明
- `node_enum.h` — 枚举双向映射
- `node_type.h` — 节点类型和 socket 类型声明
- `node_xml.h` — XML 序列化接口声明

### LIB
链接依赖库：
- `cycles_util` — Cycles 工具库，提供 ustring、array、map、MD5、Transform 等基础设施

## 依赖关系

- **构建依赖**: `cycles_util`（Cycles 工具库）
- **构建产物**: `cycles_graph` 静态库（通过 `cycles_add_library()` 自定义 CMake 函数创建）
- **被依赖**: 被 `cycles_scene`、`cycles_session`、`cycles_hydra` 等上层模块链接使用
