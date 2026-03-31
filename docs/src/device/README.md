# device - 设备抽象层

## 概述

`device/` 模块是 Cycles 渲染引擎的硬件抽象层，负责将路径追踪计算统一调度到不同的计算后端（CPU、CUDA、OptiX、HIP、HIPRT、Metal、oneAPI）上执行。该模块采用经典的工厂模式与继承体系设计：顶层 `Device` 基类定义了内存管理、内核加载、BVH 构建、命令队列等统一接口，各后端子目录提供具体实现。

核心设计目标：
- **后端透明**：上层渲染逻辑（`session/`、`integrator/`）无需感知底层硬件差异。
- **多设备协同**：通过 `MultiDevice` 支持多 GPU 并行渲染和 CPU+GPU 混合渲染。
- **延迟初始化**：设备列表和驱动程序仅在用户实际选择时才初始化，避免驱动异常导致启动崩溃。

## 目录结构

| 文件名 | 类型 | 说明 |
|--------|------|------|
| `device.h` | 头文件 | `Device`、`GPUDevice` 基类定义，`DeviceInfo`、`DeviceType` 枚举 |
| `device.cpp` | 源文件 | 设备工厂 `Device::create()`，设备枚举与能力查询 |
| `memory.h` | 头文件 | 设备内存抽象：`device_memory`、`device_vector`、`device_only_memory`、`device_texture`、`device_sub_ptr` |
| `memory.cpp` | 源文件 | 设备内存基类实现 |
| `kernel.h` | 头文件 | `DeviceKernel` 枚举辅助函数、`DeviceKernelMask` |
| `kernel.cpp` | 源文件 | 内核名称字符串转换 |
| `queue.h` | 头文件 | `DeviceQueue` 命令队列基类、`DeviceKernelArguments` 参数容器 |
| `queue.cpp` | 源文件 | 队列调试日志与性能统计 |
| `denoise.h` | 头文件 | `DenoiseParams`、`DenoiserType` 降噪参数定义 |
| `denoise.cpp` | 源文件 | 降噪参数实现 |
| `graphics_interop.h` | 头文件 | `DeviceGraphicsInterop` 图形 API 互操作基类（OpenGL/Vulkan/Metal） |
| `graphics_interop.cpp` | 源文件 | 图形互操作公共实现 |
| `CMakeLists.txt` | 构建 | CMake 构建配置 |
| `cpu/` | 子目录 | CPU 后端实现 |
| `cuda/` | 子目录 | NVIDIA CUDA 后端实现 |
| `optix/` | 子目录 | NVIDIA OptiX 光线追踪后端实现 |
| `hip/` | 子目录 | AMD HIP 后端实现 |
| `hiprt/` | 子目录 | AMD HIPRT 硬件光线追踪后端实现 |
| `metal/` | 子目录 | Apple Metal 后端实现 |
| `oneapi/` | 子目录 | Intel oneAPI/SYCL 后端实现 |
| `multi/` | 子目录 | 多设备聚合层 |
| `dummy/` | 子目录 | 空设备（错误回退） |

## 核心类与数据结构

### DeviceType 枚举

定义位置：`device.h`

```
DEVICE_NONE, DEVICE_CPU, DEVICE_CUDA, DEVICE_MULTI, DEVICE_OPTIX,
DEVICE_HIP, DEVICE_HIPRT, DEVICE_METAL, DEVICE_ONEAPI, DEVICE_DUMMY
```

### DeviceInfo

定义位置：`device.h`

设备描述信息，用于设备枚举与用户选择。关键字段：
- `type` / `id` / `description` — 设备类型标识
- `has_nanovdb` / `has_osl` / `has_guiding` — 功能支持标记
- `has_peer_memory` — P2P 显存访问能力
- `use_hardware_raytracing` — 是否启用硬件光线追踪
- `denoisers` — 支持的降噪器类型掩码
- `multi_devices` — 多设备配置时的子设备列表

### Device (基类)

定义位置：`device.h`

所有设备后端的公共基类。关键方法：

| 方法 | 说明 |
|------|------|
| `create()` | 静态工厂方法，根据 `DeviceInfo` 创建具体设备实例 |
| `available_devices()` | 延迟枚举所有可用设备 |
| `load_kernels()` | 加载/编译路径追踪内核 |
| `build_bvh()` | 构建加速结构（BVH） |
| `gpu_queue_create()` | 创建 GPU 命令队列 |
| `mem_alloc()` / `mem_copy_to()` / `mem_free()` | 设备内存管理（纯虚函数） |
| `const_copy_to()` | 将常量数据拷贝到设备全局内存 |
| `foreach_device()` | 遍历所有子设备（多设备场景） |
| `should_use_graphics_interop()` | 判断是否使用图形 API 互操作 |

### GPUDevice (GPU 公共基类)

定义位置：`device.h`

继承自 `Device`，为所有 GPU 后端（CUDA、HIP、oneAPI）提供公共功能：
- `texture_info` — 绑定式纹理信息向量
- `generic_alloc()` / `generic_free()` / `generic_copy_to()` — 通用 GPU 内存分配逻辑（含设备显存/主机映射内存回退策略）
- `move_textures_to_host()` — 显存不足时将纹理迁移到主机映射内存
- `init_host_memory()` — 初始化主机映射内存限制

### DeviceQueue (命令队列)

定义位置：`queue.h`

GPU 内核执行的命令队列抽象。关键方法：
- `enqueue()` — 入队内核执行
- `synchronize()` — 等待队列中所有内核执行完毕
- `num_concurrent_states()` — 返回积分器并发状态数
- `zero_to_device()` / `copy_to_device()` / `copy_from_device()` — 队列内存操作
- `graphics_interop_create()` — 创建图形互操作对象

### device_memory 体系

定义位置：`memory.h`

| 类 | 说明 |
|----|------|
| `device_memory` | 内存基类，管理主机/设备指针与生命周期 |
| `device_only_memory<T>` | 仅设备端内存，无主机对应 |
| `device_vector<T>` | 主机-设备交换向量，支持 `alloc()`、`copy_to_device()`、`copy_from_device()` |
| `device_texture` | 2D/3D 纹理内存 |
| `device_sub_ptr` | 已分配内存的子区域指针 |

### DenoiseParams

定义位置：`denoise.h`

降噪参数配置，支持 OptiX Denoiser 和 OpenImageDenoise (OIDN)。

## 模块架构

```
                        Device (基类)
                       /      \
                 CPUDevice    GPUDevice
                              /    |    \
                       CUDADevice HIPDevice OneapiDevice  MetalDevice
                          |          |
                     OptiXDevice  HIPRTDevice
```

继承关系说明：
- `CPUDevice` 直接继承 `Device`，使用本机 CPU 指令集（支持 AVX2 优化）。
- `GPUDevice` 为所有 GPU 后端提供通用显存管理和纹理映射。
- `CUDADevice` 继承 `GPUDevice`，`OptiXDevice` 进一步继承 `CUDADevice`（OptiX 基于 CUDA 运行时）。
- `HIPDevice` 继承 `GPUDevice`，`HIPRTDevice` 进一步继承 `HIPDevice`（HIPRT 基于 HIP 运行时）。
- `MetalDevice` 直接继承 `Device`（非 `GPUDevice`），拥有独立的内存管理实现。
- `OneapiDevice` 继承 `GPUDevice`。
- `MultiDevice` 继承 `Device`，聚合多个子设备。
- `DummyDevice` 继承 `Device`，作为创建失败时的回退。

工厂方法 `Device::create()` 根据 `DeviceInfo::type` 分发到对应的 `device_xxx_create()` 函数。

## 依赖关系

### 上游依赖（本模块依赖）

| 模块 | 用途 |
|------|------|
| `util/` | 基础工具：线程、字符串、内存对齐、类型系统 |
| `bvh/` | BVH 加速结构参数与构建 |
| `kernel/` | 内核类型枚举（`DeviceKernel`）、内核全局变量（`KernelGlobalsCPU`） |
| `graph/` | `Node` 基类（`DenoiseParams` 序列化） |

### 下游依赖（依赖本模块）

| 模块 | 用途 |
|------|------|
| `session/` | `Session` 创建设备、管理渲染队列 |
| `scene/` | 场景数据通过 `device_memory` 上传到设备 |
| `integrator/` | 路径积分器通过 `DeviceQueue` 调度内核执行 |

## 参见

- `src/device/cpu/` — CPU 后端
- `src/device/cuda/` — CUDA 后端
- `src/device/optix/` — OptiX 后端
- `src/device/hip/` — HIP 后端
- `src/device/hiprt/` — HIPRT 后端
- `src/device/metal/` — Metal 后端
- `src/device/oneapi/` — oneAPI 后端
- `src/device/multi/` — 多设备聚合
- `src/device/dummy/` — 空设备
- `src/kernel/` — 设备内核实现
- `src/bvh/` — 加速结构
