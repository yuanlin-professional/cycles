# CMakeLists.txt - util 模块构建配置

## 概述
本文件是 Cycles 渲染器 `src/util/` 工具模块的 CMake 构建配置文件，定义了源文件列表、头文件列表、系统包含目录、链接库，以及条件编译选项。最终通过 `cycles_add_library` 构建为 `cycles_util` 静态库。

## 类与结构体
不适用（构建配置文件）。

## 核心函数
不适用。以下为构建配置的关键部分。

### 包含目录
| 变量 | 内容 |
|------|------|
| `INC` | `..`（即 `src/` 目录，使 `#include "util/xxx.h"` 可用） |
| `INC_SYS` | `${ZSTD_INCLUDE_DIRS}`（Zstandard 压缩库） |

### 源文件（SRC）- 编译单元
共 17 个 `.cpp` 文件：
- **内存管理**: `aligned_malloc.cpp`, `guarded_allocator.cpp`（在 SRC_HEADERS 中）
- **字符串与 I/O**: `string.cpp`, `path.cpp`, `log.cpp`
- **数学**: `math_cdf.cpp`, `murmurhash.cpp`, `md5.cpp`, `transform.cpp`, `transform_avx2.cpp`
- **体积数据**: `nanovdb.cpp`, `openvdb.cpp`
- **光域**: `ies.cpp`
- **系统**: `system.cpp`, `debug.cpp`, `profiling.cpp`, `task.cpp`, `thread.cpp`, `time.cpp`, `windows.cpp`

### 头文件（SRC_HEADERS）
共 90+ 个头文件，覆盖：
- 数据类型系统（`types*.h`）
- 数学运算（`math*.h`）
- 容器与算法（`vector.h`, `array.h`, `map.h`, `set.h`, `queue.h` 等）
- 并发与任务（`thread.h`, `task.h`, `tbb.h`, `semaphore.h`, `atomic.h`）
- 图像与纹理（`image.h`, `texture.h`, `half.h`, `color.h`）
- 外部库集成（`openimagedenoise.h`, `guiding.h`, `nanovdb.h`, `openvdb.h`, `param.h`, `args.h`）

### 链接库（LIB）
| 库 | 说明 |
|------|------|
| `${TBB_LIBRARIES}` | Intel TBB 并行库 |
| `${ZSTD_LIBRARIES}` | Zstandard 压缩库 |
| `${OPENVDB_LIBRARIES}` | OpenVDB（条件：`WITH_OPENVDB`） |

### 条件编译

#### AVX2 优化
```cmake
if(CXX_HAS_AVX2)
  set_source_files_properties(transform_avx2.cpp PROPERTIES COMPILE_FLAGS "${CYCLES_AVX2_FLAGS}")
endif()
```
仅对 `transform_avx2.cpp` 启用 AVX2 编译标志。

#### OpenVDB 支持
```cmake
if(WITH_OPENVDB)
  list(APPEND LIB ${OPENVDB_LIBRARIES})
endif()
```
并针对 MSVC Clang 编译器添加 `-fno-delayed-template-parsing` 以解决模板解析问题（参见 Blender #120317）。

### 构建目标
```cmake
cycles_add_library(cycles_util "${LIB}" ${SRC} ${SRC_HEADERS})
```
生成 `cycles_util` 静态库。

## 依赖关系
- **被依赖**: 几乎所有 Cycles 模块都链接 `cycles_util`
- **依赖库**: TBB, Zstandard, OpenVDB（可选）

## 关联文件
- `src/CMakeLists.txt` - 上级构建配置
- `src/cmake/macros.cmake` - `cycles_add_library` 宏定义
