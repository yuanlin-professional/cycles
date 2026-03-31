# graphics_interop.h / graphics_interop.cpp - CUDA 图形 API 互操作

## 概述

本文件实现了 CUDA 与图形 API（OpenGL、Vulkan）之间的缓冲区共享互操作功能。通过 `CUDADeviceGraphicsInterop` 类，Cycles 渲染器可以将 CUDA 计算的渲染结果直接写入图形 API 拥有的缓冲区，避免了主机端的数据中转拷贝，从而实现高效的视口实时预览渲染。该类针对 OpenGL 和 Vulkan 两种图形后端提供不同的资源注册和内存映射策略。

## 类与结构体

### `CUDADeviceGraphicsInterop`
- **继承**: `DeviceGraphicsInterop`（定义于 `device/graphics_interop.h`）
- **功能**: 管理 CUDA 与图形 API 之间的缓冲区映射与生命周期
- **禁用操作**: 拷贝构造、移动构造、拷贝赋值、移动赋值均被 `delete` 禁用
- **关键成员**:
  - `queue_` (`CUDADeviceQueue*`) — 关联的 CUDA 命令队列
  - `device_` (`CUDADevice*`) — 关联的 CUDA 设备
  - `buffer_size_` (`size_t`) — 缓冲区大小（字节）
  - `need_zero_` (`bool`) — 标记缓冲区是否需要清零
  - `cu_graphics_resource_` (`CUgraphicsResource`) — OpenGL 互操作注册的 CUDA 图形资源
  - `cu_external_memory_` (`CUexternalMemory`) — Vulkan 互操作导入的外部内存句柄
  - `cu_external_memory_ptr_` (`CUdeviceptr`) — Vulkan 外部内存映射的设备端指针
  - `vulkan_windows_handle_` (`int64_t`, 仅 Windows) — Vulkan Win32 句柄（需手动关闭）
- **关键方法**:
  - `set_buffer()` — 注册图形 API 缓冲区到 CUDA
  - `map()` — 将图形缓冲区映射为 CUDA 设备指针，供内核写入
  - `unmap()` — 取消映射，将缓冲区控制权归还图形 API
  - `free()` — 释放所有 CUDA 互操作资源

## 核心函数

### `set_buffer(GraphicsInteropBuffer &interop_buffer)`
根据图形后端类型执行不同的缓冲区注册操作：
- **OpenGL**: 调用 `cuGraphicsGLRegisterBuffer()` 将 OpenGL 缓冲区注册为 CUDA 图形资源
- **Vulkan**:
  - 构造 `CUDA_EXTERNAL_MEMORY_HANDLE_DESC`，Windows 使用 `CU_EXTERNAL_MEMORY_HANDLE_TYPE_OPAQUE_WIN32`，Linux 使用 `CU_EXTERNAL_MEMORY_HANDLE_TYPE_OPAQUE_FD`
  - 调用 `cuImportExternalMemory()` 导入外部内存
  - 调用 `cuExternalMemoryGetMappedBuffer()` 获取设备端映射指针
  - 注意：Linux 上 `cuImportExternalMemory()` 会接管文件描述符的所有权，而 Windows 上不会接管句柄

### `map()`
- **OpenGL**: 调用 `cuGraphicsMapResources()` 映射资源，再通过 `cuGraphicsResourceGetMappedPointer()` 获取设备指针
- **Vulkan**: 直接返回已缓存的 `cu_external_memory_ptr_`（Vulkan 缓冲区始终保持映射状态）
- 如果 `need_zero_` 为 true，使用 `cuMemsetD8Async()` 异步清零缓冲区

### `unmap()`
仅对 OpenGL 资源调用 `cuGraphicsUnmapResources()` 取消映射。Vulkan 资源无需显式取消映射。

### `free()`
按顺序释放所有资源：
1. 注销 OpenGL 图形资源（`cuGraphicsUnregisterResource()`）
2. 释放 Vulkan 外部内存映射指针（`cuMemFree()`）
3. 销毁 Vulkan 外部内存句柄（`cuDestroyExternalMemory()`）
4. 在 Windows 上关闭 Vulkan Win32 句柄（`CloseHandle()`）

## 依赖关系

- **内部头文件**:
  - `device/graphics_interop.h` — 基类 `DeviceGraphicsInterop`
  - `device/cuda/device_impl.h` — `CUDADevice` 类
  - `device/cuda/util.h` — `CUDAContextScope`、`cuda_device_assert` 宏
  - `session/display_driver.h` — `GraphicsInteropBuffer`、`GraphicsInteropDevice` 类型定义
  - `util/windows.h`（Windows 平台）/ `<unistd.h>`（Linux 平台）
- **被引用**:
  - `src/device/cuda/queue.cpp` — `CUDADeviceQueue::graphics_interop_create()` 创建本类实例

## 实现细节 / 关键算法

1. **平台差异处理**: Vulkan 外部内存句柄在 Windows 和 Linux 上的行为不同。Windows 使用 Win32 不透明句柄（`HANDLE`），CUDA 不会接管所有权，需由调用方手动关闭。Linux 使用文件描述符（fd），CUDA 导入时会接管所有权，导入失败时需手动 `close()` fd。
2. **异步操作**: `map()` 中的缓冲区清零使用 `cuMemsetD8Async()` 在 CUDA 流上异步执行，与内核执行流水线化。映射和取消映射操作也通过队列的 `stream()` 进行，保证操作顺序。
3. **所有权语义**: 通过禁用拷贝和移动语义确保 CUDA 资源的唯一所有权，防止双重释放。
4. **惰性清零**: `need_zero_` 标志支持跨帧累积清零请求（通过 `|=` 运算符），在实际 `map()` 时一次性执行。

## 关联文件

- `src/device/cuda/queue.h` / `queue.cpp` — 命令队列创建图形互操作实例
- `src/device/cuda/device_impl.h` / `device_impl.cpp` — CUDA 设备（`should_use_graphics_interop()` 决定是否启用互操作）
- `src/device/graphics_interop.h` — 基类定义
- `src/session/display_driver.h` — 图形互操作缓冲区和设备类型定义
- `src/integrator/path_trace_work_gpu.h` — 路径追踪 GPU 工作使用图形互操作
