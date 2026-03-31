# CMakeLists.txt - BVH 子系统构建配置

## 概述

本文件是 Cycles 渲染器 `src/bvh/` 目录的 CMake 构建配置文件。它定义了 BVH 子系统的源文件列表、头文件列表、包含目录、链接库以及条件编译选项,最终通过 `cycles_add_library()` 宏生成 `cycles_bvh` 静态库。该库是 Cycles 渲染器加速结构的核心组件。

## 构建目标

### cycles_bvh（静态库）
通过 `cycles_add_library(cycles_bvh "${LIB}" ${SRC} ${SRC_HEADERS})` 生成。

## 源文件列表

### SRC（C++ 源文件）
| 文件 | 功能 |
|------|------|
| `octree.cpp` | 八叉树实现 |
| `bvh.cpp` | BVH 基类和工厂方法 |
| `bvh2.cpp` | 二叉 BVH 实现 |
| `binning.cpp` | 表面积启发式(SAH)分箱器 |
| `build.cpp` | BVH 树构建器 |
| `embree.cpp` | Embree 后端集成 |
| `hiprt.cpp` | HIPRT 后端集成 |
| `multi.cpp` | 多后端组合 BVH |
| `node.cpp` | BVH 节点类型 |
| `optix.cpp` | OptiX 后端集成 |
| `sort.cpp` | 图元排序 |
| `split.cpp` | 空间分割策略 |
| `unaligned.cpp` | 非轴对齐包围盒启发式 |

### SRC_METAL（Metal 源文件，条件编译）
| 文件 | 功能 |
|------|------|
| `metal.mm` | Metal 后端工厂接口（Objective-C++） |

### SRC_HEADERS（头文件）
| 文件 | 功能 |
|------|------|
| `octree.h` | 八叉树头文件 |
| `bvh.h` | BVH 基类定义 |
| `bvh2.h` | 二叉 BVH 定义 |
| `binning.h` | 分箱器定义 |
| `build.h` | 构建器定义 |
| `embree.h` | Embree 后端定义 |
| `hiprt.h` | HIPRT 后端定义 |
| `multi.h` | 多后端 BVH 定义 |
| `node.h` | 节点类型定义 |
| `optix.h` | OptiX 后端定义 |
| `params.h` | BVH 参数和引用类型 |
| `sort.h` | 排序头文件 |
| `split.h` | 分割策略头文件 |
| `unaligned.h` | 非轴对齐头文件 |
| `metal.h` | Metal 后端头文件 |

## 包含目录

- **INC**: `..`（即 `src/` 目录,允许以 `bvh/xxx.h` 形式引用头文件）
- **INC_SYS**: 空（无系统级包含目录）

## 链接库

### 基础链接库
- `cycles_scene` — Cycles 场景管理库
- `cycles_util` — Cycles 工具库

### 条件链接库
- **WITH_CYCLES_EMBREE**:
  - `${EMBREE_LIBRARIES}` — Intel Embree 库
  - `${SYCL_LIBRARIES}` — Intel SYCL 库（仅当 `EMBREE_SYCL_SUPPORT` 启用时）

## 条件编译

### Metal 后端
```cmake
if(WITH_CYCLES_DEVICE_METAL)
  list(APPEND SRC ${SRC_METAL})
  add_definitions(-DWITH_METAL)
endif()
```
当启用 Metal 设备支持时:
1. 将 `metal.mm` 添加到源文件列表
2. 定义 `WITH_METAL` 预处理宏

### Embree 后端
```cmake
if(WITH_CYCLES_EMBREE)
  list(APPEND LIB ${EMBREE_LIBRARIES})
  if(EMBREE_SYCL_SUPPORT)
    list(APPEND LIB ${SYCL_LIBRARIES})
  endif()
endif()
```
当启用 Embree 支持时链接 Embree 库。若 Embree 带有 SYCL GPU 支持,还需链接 SYCL 库。

## 实现细节

### 编译宏与条件编译
各后端的源文件（`embree.cpp`、`hiprt.cpp`、`optix.cpp`）始终编译,但其内部代码受各自的条件编译宏保护（`WITH_EMBREE`、`WITH_HIPRT`、`WITH_OPTIX`）。只有 `metal.mm` 在 CMake 层面按条件添加,因为 Objective-C++ 文件在非 macOS 平台上可能无法编译。

### cycles_add_library 宏
这是 Cycles 项目自定义的 CMake 宏,负责创建静态库并配置标准编译选项。第二个参数 `${LIB}` 指定要链接的依赖库。

### 库依赖方向
`cycles_bvh` 依赖 `cycles_scene`（因为 BVH 构建需要访问 Mesh、Hair、Object 等场景类）和 `cycles_util`（基础工具类）。反过来,场景管理代码也会调用 BVH 构建接口,形成双向依赖,由 CMake 的链接器处理。

## 关联文件

- `src/bvh/` 下的所有 `.h`、`.cpp` 和 `.mm` 文件
- `src/scene/` — `cycles_scene` 库
- `src/util/` — `cycles_util` 库
- 项目根目录的 `CMakeLists.txt` — 顶层构建配置
