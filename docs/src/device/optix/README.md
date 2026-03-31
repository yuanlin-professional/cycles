# device/optix - NVIDIA OptiX 硬件光线追踪设备实现

## 概述

`optix/` 子目录实现了 Cycles 的 NVIDIA OptiX 硬件光线追踪后端。`OptiXDevice` 继承自 `CUDADevice`，在 CUDA 后端的基础上集成 NVIDIA OptiX SDK，利用 NVIDIA RTX GPU 的 RT Core 硬件单元加速光线-场景求交。

核心特点：
- **RT Core 加速**：利用 NVIDIA GPU 的专用光线追踪硬件单元（RT Core），显著提升光线遍历性能。
- **丰富的程序组**：定义了 Ray Generation、Miss、Hit、Callable 共 `NUM_PROGRAM_GROUPS`（50+）个程序组，覆盖所有光线类型和几何图元。
- **双流水线架构**：分为 `PIP_SHADE`（着色流水线）和 `PIP_INTERSECT`（求交流水线）。
- **OSL 支持**：OptiX 后端支持在 GPU 上运行 Open Shading Language。
- **内置曲线与点云**：通过 OptiX 内置模块支持硬件加速的曲线和点云图元求交。
- **SBT（Shader Binding Table）**：通过着色器绑定表将程序组关联到几何实例。

## 目录结构

| 文件名 | 类型 | 说明 |
|--------|------|------|
| `device.h` | 头文件 | `device_optix_init()`、`device_optix_create()`、`device_optix_info()` 工厂函数 |
| `device.cpp` | 源文件 | OptiX 设备初始化与枚举（基于 CUDA 设备列表过滤） |
| `device_impl.h` | 头文件 | `OptiXDevice` 类定义、程序组/流水线枚举、`SbtRecord` 结构 |
| `device_impl.cpp` | 源文件 | `OptiXDevice` 实现：OptiX 上下文创建、模块编译、流水线构建、BVH 构建 |
| `queue.h` | 头文件 | `OptiXDeviceQueue` 命令队列类 |
| `queue.cpp` | 源文件 | OptiX 内核通过 `optixLaunch` 执行 |
| `util.h` | 头文件 | OptiX 错误检查宏 `optix_assert`、头文件包含管理 |

## 核心类与数据结构

### OptiXDevice

定义位置：`device_impl.h`

继承自 `CUDADevice`。关键成员：

| 成员 | 说明 |
|------|------|
| `context` | OptiX 设备上下文 (`OptixDeviceContext`) |
| `optix_module` | OptiX 主模块（包含所有内核） |
| `builtin_modules[4]` | 内置模块（曲线、点云等内置图元支持） |
| `pipelines[NUM_PIPELINES]` | 两条流水线：着色 (`PIP_SHADE`) 和求交 (`PIP_INTERSECT`) |
| `groups[NUM_PROGRAM_GROUPS]` | 所有 OptiX 程序组 |
| `pipeline_options` | 流水线编译选项 |
| `sbt_data` | 着色器绑定表数据 |
| `launch_params` | 内核启动参数（设备端） |
| `tlas_handle` | 顶层加速结构遍历句柄 |
| `osl_globals` / `osl_modules` / `osl_groups` | OSL 支持（条件编译 `WITH_OSL`） |

### 程序组枚举

定义位置：`device_impl.h`

OptiX 程序组按光线类型分组：

**Ray Generation（光线生成）**：
- `PG_RGEN_INTERSECT_CLOSEST` — 最近相交
- `PG_RGEN_INTERSECT_SHADOW` — 阴影射线
- `PG_RGEN_INTERSECT_SUBSURFACE` — 次表面散射
- `PG_RGEN_INTERSECT_VOLUME_STACK` — 体积栈
- `PG_RGEN_SHADE_SURFACE` / `PG_RGEN_SHADE_VOLUME` / `PG_RGEN_SHADE_SHADOW` 等 — 各着色阶段

**Hit（命中）**：
- `PG_HITD` / `PG_HITS` / `PG_HITL` / `PG_HITV` — 默认/阴影/局部/体积命中组
- 对应的运动模糊（`_MOTION`）、线性曲线（`_CURVE_LINEAR`）、带状曲线（`_CURVE_RIBBON`）、点云（`_POINTCLOUD`）变体

**Callable（可调用程序）**：
- `PG_CALL_SVM_AO` — 环境光遮蔽
- `PG_CALL_SVM_BEVEL` — 斜面法线

### 流水线枚举

- `PIP_SHADE` — 着色流水线
- `PIP_INTERSECT` — 求交流水线

### OptiXDeviceQueue

定义位置：`queue.h`

继承自 `CUDADeviceQueue`。重写 `enqueue()` 方法：
- 对于求交相关内核（`DEVICE_KERNEL_INTEGRATOR_INTERSECT_*`），使用 `optixLaunch` 启动 OptiX 光线追踪。
- 对于其他内核，回退到 CUDA 常规内核启动。

### SbtRecord

定义位置：`device_impl.h`

OptiX 着色器绑定表记录，包含 `OPTIX_SBT_RECORD_HEADER_SIZE` 字节的头部数据。

## 硬件要求

- **GPU**：NVIDIA GPU，支持 OptiX（Turing/RTX 20 系列及更新，即具有 RT Core 的 GPU）
- **驱动**：支持 OptiX 的 NVIDIA 驱动
- **OptiX SDK**：编译时需要 OptiX SDK 头文件（`optix_stubs.h`）
- **CUDA**：底层依赖 CUDA Driver API（继承自 CUDADevice）
- **降噪器**：支持 OptiX Denoiser 和 OpenImageDenoise (OIDN)
- **条件编译**：`WITH_OPTIX` 和 `WITH_CUDA`

## API 封装

- **OptiX API**：`optixDeviceContextCreate`、`optixModuleCreate`、`optixPipelineCreate`、`optixProgramGroupCreate`、`optixAccelBuild`、`optixLaunch`
- **CUDA Driver API**：继承自 `CUDADevice`
- 错误处理：`optix_assert` / `optix_device_assert` 宏

## 依赖关系

### 上游依赖（本模块依赖）

| 模块 | 用途 |
|------|------|
| `device/cuda/` | `CUDADevice` 基类、`CUDADeviceQueue` 基类 |
| `device/` | 设备抽象、内存体系 |
| `kernel/` | 内核类型枚举、OSL 全局变量 |
| `bvh/` | BVH 参数、BVHOptiX 数据结构 |
| `util/` | 任务池、日志 |
| OptiX SDK | OptiX 光线追踪 API |
| CUEW / CUDA SDK | CUDA Driver API |

### 下游依赖（依赖本模块）

| 模块 | 用途 |
|------|------|
| `device/device.cpp` | 工厂方法调用 `device_optix_create()` |

## 参见

- `src/device/cuda/` — CUDA 后端基类
- `src/device/hiprt/` — 类似的 AMD 硬件光线追踪实现
- `src/bvh/` — BVH 加速结构
- `src/kernel/device/optix/` — OptiX 内核实现（PTX）
