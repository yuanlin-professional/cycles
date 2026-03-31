# CMakeLists.txt - session 模块构建配置

## 概述
本文件是 Cycles 渲染器 `session` 模块的 CMake 构建脚本，负责定义该模块的源文件、头文件、包含路径和库依赖，并通过 `cycles_add_library()` 宏将其编译为 `cycles_session` 库。session 模块是 Cycles 的渲染会话管理层，包含渲染缓冲区、降噪管线、显示驱动、图像合并、瓦片管理等核心功能。

## 构建配置

### 包含路径
- **内部包含路径（INC）**: `..`（即 `src/` 目录，允许以 `session/`、`device/`、`scene/` 等相对路径引用其他模块头文件）
- **系统包含路径（INC_SYS）**: 无额外系统路径

### 源文件（SRC）
| 文件 | 功能 |
|------|------|
| `buffers.cpp` | 渲染缓冲区管理与通道数据布局 |
| `display_driver.cpp` | 图形互操作缓冲区实现 |
| `denoising.cpp` | 独立降噪管线与图像帧处理 |
| `merge.cpp` | 多层 EXR 图像合并 |
| `session.cpp` | 渲染会话控制与生命周期管理 |
| `tile.cpp` | 瓦片管理与磁盘分块读写 |

### 头文件（SRC_HEADERS）
| 文件 | 功能 |
|------|------|
| `buffers.h` | 渲染缓冲区类与通道参数定义 |
| `display_driver.h` | 显示驱动抽象接口与图形互操作类型 |
| `denoising.h` | 降噪管线类与图像数据结构 |
| `merge.h` | 图像合并器接口 |
| `output_driver.h` | 离线渲染输出驱动接口（仅头文件） |
| `session.h` | 渲染会话类与参数定义 |
| `tile.h` | 瓦片类与瓦片管理器定义 |

### 库依赖（LIB）
| 库名 | 说明 |
|------|------|
| `cycles_device` | 设备抽象层（CPU、GPU 等计算设备管理） |
| `cycles_integrator` | 积分器模块（路径追踪、降噪器、渲染调度） |
| `cycles_util` | 工具库（数学、字符串、线程、日志等基础设施） |

### 构建命令
- `include_directories(${INC})` — 添加内部包含路径
- `include_directories(SYSTEM ${INC_SYS})` — 添加系统包含路径（抑制第三方头文件警告）
- `cycles_add_library(cycles_session "${LIB}" ${SRC} ${SRC_HEADERS})` — 使用 Cycles 自定义宏创建 `cycles_session` 库目标，自动处理依赖链接

## 依赖关系
- **上游依赖**: `cycles_device`、`cycles_integrator`、`cycles_util`
- **隐式外部依赖**: OpenImageIO（被 `denoising.cpp`、`merge.cpp`、`tile.cpp` 使用）
- **下游被依赖**: `cycles_session` 库被 Cycles 主应用和 Hydra 渲染委托等链接使用

## 关联文件
- `src/CMakeLists.txt` — 顶层构建脚本，包含本模块
- `src/device/CMakeLists.txt` — 设备模块构建配置
- `src/integrator/CMakeLists.txt` — 积分器模块构建配置
- `src/util/CMakeLists.txt` — 工具库构建配置
