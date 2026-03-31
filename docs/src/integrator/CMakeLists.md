# CMakeLists.txt - 积分器模块构建配置

## 概述

本文件是 Cycles 渲染器 `integrator` 模块的 CMake 构建脚本，定义了该模块包含的源文件、头文件、依赖库以及条件编译选项。最终将所有文件编译为 `cycles_integrator` 静态库。

## 构建内容

### 包含路径

- `INC`: 上级目录 `..`（即 `src/`），用于解析 `integrator/xxx.h`、`session/xxx.h` 等内部头文件引用
- `INC_SYS`: 系统包含路径（当前为空）

### 源文件列表 (SRC)

| 文件 | 功能 |
|------|------|
| `adaptive_sampling.cpp` | 自适应采样逻辑 |
| `denoiser.cpp` | 降噪器基类 |
| `denoiser_gpu.cpp` | GPU 降噪器基类 |
| `denoiser_oidn.cpp` | OpenImageDenoise 降噪器 |
| `denoiser_oidn_gpu.cpp` | OpenImageDenoise GPU 降噪器 |
| `denoiser_optix.cpp` | OptiX 降噪器 |
| `path_trace.cpp` | 路径追踪核心 |
| `tile.cpp` | 图块大小计算 |
| `pass_accessor.cpp` | Pass 数据访问器基类 |
| `pass_accessor_cpu.cpp` | CPU Pass 数据访问器 |
| `pass_accessor_gpu.cpp` | GPU Pass 数据访问器 |
| `path_trace_display.cpp` | 渲染显示管理 |
| `path_trace_tile.cpp` | 图块数据接口 |
| `path_trace_work.cpp` | 设备工作基类 |
| `path_trace_work_cpu.cpp` | CPU 设备工作实现 |
| `path_trace_work_gpu.cpp` | GPU 设备工作实现 |
| `render_scheduler.cpp` | 渲染调度器 |
| `shader_eval.cpp` | 着色器求值 |
| `work_balancer.cpp` | 多设备负载均衡 |
| `work_tile_scheduler.cpp` | GPU 图块调度器 |

### 头文件列表 (SRC_HEADERS)

与源文件一一对应的头文件，另外包含:
- `guiding.h` — 路径引导相关头文件（无对应 .cpp）

### 依赖库 (LIB)

| 库 | 说明 |
|----|------|
| `cycles_device` | 设备抽象层 |
| `cycles_session` | 会话管理（注：存在循环依赖，因为需要访问 `RenderBuffers`） |
| `cycles_util` | 工具库 |

### 条件依赖

- **`WITH_OPENIMAGEDENOISE`**: 链接 `${OPENIMAGEDENOISE_LIBRARIES}`（Intel Open Image Denoise 库）
- **`WITH_CYCLES_PATH_GUIDING`**: 链接 `${OPENPGL_LIBRARIES}`（Intel Open Path Guiding Library）

### 构建目标

```cmake
cycles_add_library(cycles_integrator "${LIB}" ${SRC} ${SRC_HEADERS})
```

生成 `cycles_integrator` 静态库，包含所有源文件和头文件。

## 依赖关系

- **内部依赖**: `cycles_device`, `cycles_session`, `cycles_util`
- **可选外部依赖**: OpenImageDenoise（降噪）, OpenPGL（路径引导）
- **被依赖**: 上层 Cycles 构建系统链接此库

## 实现细节

1. **循环依赖注意**: 注释中提到 `cycles_session` 的依赖是因为需要访问 `RenderBuffers`，存在一定的循环依赖。将来可能通过调整文件结构来消除。

2. **条件编译**: 降噪和路径引导功能通过 CMake 选项控制，不启用时对应的库不链接，相关代码通过预处理宏（`WITH_OPENIMAGEDENOISE`、`WITH_PATH_GUIDING`）条件编译。

## 关联文件

- `src/CMakeLists.txt` — 上层构建脚本
- `src/device/CMakeLists.txt` — 设备层构建
- `src/session/CMakeLists.txt` — 会话层构建
