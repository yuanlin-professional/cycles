# device/oneapi - Intel oneAPI/SYCL 设备实现

## 概述

`oneapi/` 子目录实现了 Cycles 的 Intel oneAPI GPU 渲染后端。`OneapiDevice` 继承自 `GPUDevice`，通过 SYCL（基于 Intel 的 DPC++ 实现）和 Level Zero 运行时在 Intel GPU 上执行路径追踪内核。

核心特点：
- **SYCL 编程模型**：使用 SYCL 作为设备端编程接口，通过 USM（Unified Shared Memory）管理内存。
- **Embree GPU 集成**：可选使用 Intel Embree GPU 后端进行硬件加速的光线求交（通过 `RTCDevice` 在 GPU 上运行）。
- **原子排序**：支持本地原子排序内核 (`supports_local_atomic_sort()` 返回 true)。
- **图形互操作**：支持与 Vulkan 的图形资源互操作（条件编译 `SYCL_LINEAR_MEMORY_INTEROP_AVAILABLE`）。
- **多 dGPU 检测**：检测多 Intel 独立 GPU 配置并启用兼容性解决方案。

## 目录结构

| 文件名 | 类型 | 说明 |
|--------|------|------|
| `device.h` | 头文件 | `device_oneapi_init()`、`device_oneapi_create()`、`device_oneapi_info()`、`device_oneapi_capabilities()` 工厂函数 |
| `device.cpp` | 源文件 | oneAPI 设备枚举与初始化 |
| `device_impl.h` | 头文件 | `OneapiDevice` 类定义 |
| `device_impl.cpp` | 源文件 | `OneapiDevice` 实现：SYCL 队列管理、USM 内存管理、Embree GPU 集成 |
| `queue.h` | 头文件 | `OneapiDeviceQueue` 命令队列类 |
| `queue.cpp` | 源文件 | oneAPI 内核入队与同步 |
| `graphics_interop.h` | 头文件 | `OneapiDeviceGraphicsInterop` 图形互操作类 |
| `graphics_interop.cpp` | 源文件 | Vulkan 外部内存导入与映射 |

## 核心类与数据结构

### OneapiDevice

定义位置：`device_impl.h`

继承自 `GPUDevice`。关键成员：

| 成员 | 说明 |
|------|------|
| `device_queue_` | SYCL 队列指针 (`SyclQueue*`) |
| `embree_device` | Embree GPU 设备（条件编译 `WITH_EMBREE_GPU`） |
| `embree_traversable` | Embree GPU 场景遍历句柄 |
| `const_mem_map_` | 常量内存映射表 |
| `kg_memory_` / `kg_memory_device_` | 内核全局变量（主机/设备端） |
| `max_memory_on_device_` | 设备最大可用内存 |
| `use_hardware_raytracing` | 是否使用硬件光线追踪（Embree GPU） |
| `is_several_intel_dgpu_devices_detected` | 多 Intel dGPU 兼容性标记 |

关键方法：

| 方法 | 说明 |
|------|------|
| `load_kernels()` | 加载 oneAPI 内核 |
| `build_bvh()` | 构建 Embree GPU 加速结构（条件编译 `WITH_EMBREE_GPU`） |
| `mem_alloc()` / `mem_free()` | USM 内存管理 |
| `host_alloc()` / `host_free()` | USM 主机端分配 |
| `usm_aligned_alloc_host()` / `usm_alloc_device()` / `usm_free()` | 底层 USM 操作 |
| `set_global_memory()` | 设置内核全局内存指针 |
| `enqueue_kernel()` | 底层内核入队 |
| `get_adjusted_global_and_local_sizes()` | 计算最优工作组大小 |
| `iterate_devices()` | 静态方法，枚举所有 SYCL 设备 |
| `architecture_information()` | 静态方法，查询设备架构与优化状态 |

### OneapiDeviceQueue

定义位置：`queue.h`

继承自 `DeviceQueue`。关键特性：
- `enqueue()` — 通过 `OneapiDevice::enqueue_kernel()` 执行设备内核
- `supports_local_atomic_sort()` — 始终返回 true
- `num_sort_partitions()` — 针对 oneAPI 设备优化的排序分区数
- `kernel_context_` — 内核上下文，封装 SYCL 队列与内核全局变量

### OneapiDeviceGraphicsInterop

定义位置：`graphics_interop.h`

通过 SYCL oneAPI 实验性扩展（`sycl::ext::oneapi::experimental::external_mem`）实现 Vulkan 外部内存导入，用于视口零拷贝渲染。

## 硬件要求

- **GPU**：Intel Arc 系列独立 GPU（如 A770、A750），或集成 GPU（Xe 系列）
- **驱动**：Intel GPU 驱动（Linux: Intel Compute Runtime / Level Zero；Windows: Intel Graphics Driver）
- **oneAPI 运行时**：需要 Intel oneAPI DPC++/C++ 编译器和运行时
- **Embree GPU**：需要支持 GPU 的 Embree 4.x（条件编译 `WITH_EMBREE_GPU`）
- **条件编译**：`WITH_ONEAPI`

## API 封装

- **SYCL API**（通过 Intel DPC++）：
  - 队列管理：`sycl::queue`
  - 内存管理：USM（`sycl::malloc_device`、`sycl::malloc_host`、`sycl::free`）
  - 内核执行：`sycl::queue::submit`
  - 外部内存：`sycl::ext::oneapi::experimental::external_mem`
- **Level Zero**：底层 GPU 运行时
- **Embree C API**：`rtcNewDevice`、`rtcNewScene`（GPU 变体）

## 依赖关系

### 上游依赖（本模块依赖）

| 模块 | 用途 |
|------|------|
| `device/` | `GPUDevice` 基类、`DeviceQueue` 基类、内存体系 |
| `kernel/device/oneapi/` | oneAPI 内核实现与 `KernelContext` |
| `kernel/` | 内核类型枚举 |
| `util/` | 线程、日志、映射表 |
| Intel oneAPI / DPC++ | SYCL 运行时 |
| Embree (GPU) | 光线追踪加速（可选） |

### 下游依赖（依赖本模块）

| 模块 | 用途 |
|------|------|
| `device/device.cpp` | 工厂方法调用 `device_oneapi_create()` |

## 参见

- `src/device/device.h` — `GPUDevice` 基类
- `src/device/cuda/` — 结构类似的 CUDA 后端
- `src/kernel/device/oneapi/` — oneAPI 内核实现
