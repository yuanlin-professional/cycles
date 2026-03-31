# graphics_interop.h / graphics_interop.cpp - HIP 图形 API 互操作

## 概述

本文件实现了 HIP 设备与图形 API（如 OpenGL）之间的资源互操作功能。通过 `HIPDeviceGraphicsInterop` 类，Cycles 渲染结果可以直接写入图形 API 的缓冲区，避免额外的内存拷贝，从而提升视口实时预览的性能。当前实现仅支持 OpenGL 互操作，Vulkan 和 Metal 支持尚未实现。

## 类与结构体

### `HIPDeviceGraphicsInterop`
- **继承**: `DeviceGraphicsInterop`（`device/graphics_interop.h` 中定义的图形互操作基类）
- **功能**: 管理 HIP 与图形 API 之间的缓冲区共享，负责资源注册、映射和取消映射
- **关键成员**:
  - `HIPDeviceQueue *queue_` — 关联的 HIP 设备队列
  - `HIPDevice *device_` — 关联的 HIP 设备（从队列中获取）
  - `size_t buffer_size_` — 缓冲区大小（字节）
  - `bool need_zero_` — 标记缓冲区是否需要清零
  - `hipGraphicsResource hip_graphics_resource_` — HIP 图形资源句柄（用于 OpenGL 互操作）
  - `hipDeviceptr_t hip_external_memory_ptr_` — 外部内存设备指针（用于 Vulkan 互操作，当前未实现）
- **关键方法**:
  - `set_buffer()` — 注册图形 API 缓冲区
  - `map()` — 映射缓冲区到 HIP 设备地址空间
  - `unmap()` — 取消映射
  - `free()` — 释放所有互操作资源
- **拷贝语义**: 显式禁用拷贝构造、移动构造、拷贝赋值和移动赋值操作符

## 核心函数

### `HIPDeviceGraphicsInterop::set_buffer()`
- **参数**: `GraphicsInteropBuffer &interop_buffer`
- **功能**: 根据图形 API 类型注册缓冲区：
  - 空缓冲区时调用 `free()` 释放已有资源
  - 累积清零请求（`need_zero_ |= interop_buffer.take_zero()`）
  - 若缓冲区句柄未变更则直接返回
  - OpenGL 类型：调用 `hipGraphicsGLRegisterBuffer()` 注册 GL 缓冲区
  - Vulkan / Metal / NONE：当前为空实现（TODO）

### `HIPDeviceGraphicsInterop::map()`
- **返回值**: `device_ptr` — 映射后的 HIP 设备指针
- **功能**:
  - 对 OpenGL 资源：调用 `hipGraphicsMapResources()` 和 `hipGraphicsResourceGetMappedPointer()` 获取设备指针
  - 对 Vulkan 资源（预留）：直接使用 `hip_external_memory_ptr_`（常驻映射）
  - 若 `need_zero_` 为真，使用 `hipMemsetD8Async()` 异步清零缓冲区

### `HIPDeviceGraphicsInterop::unmap()`
- **功能**: 对 OpenGL 资源调用 `hipGraphicsUnmapResources()` 取消映射，使缓冲区重新可被图形 API 使用

### `HIPDeviceGraphicsInterop::free()`
- **功能**: 清理所有互操作资源：
  - 注销 OpenGL 图形资源 (`hipGraphicsUnregisterResource()`)
  - 释放外部内存指针 (`hipFree()`)
  - 重置缓冲区大小和清零标志

## 依赖关系
- **内部头文件**:
  - `device/graphics_interop.h` — `DeviceGraphicsInterop` 基类
  - `session/display_driver.h` — 显示驱动定义
  - `device/hip/device_impl.h` — `HIPDevice` 类定义
  - `device/hip/util.h` — `HIPContextScope`、`hip_device_assert` 宏
  - `hipew.h` — HIP 动态加载包装（条件编译 `WITH_HIP_DYNLOAD`）
- **被引用**: `src/device/hip/queue.cpp`（`HIPDeviceQueue::graphics_interop_create()` 中创建实例）

## 实现细节 / 关键算法

- **上下文安全**: 所有 HIP API 调用都包裹在 `HIPContextScope` 中，确保在正确的设备上下文中执行。
- **异步清零**: 缓冲区清零操作通过 `hipMemsetD8Async()` 在设备队列的流上异步执行，避免阻塞 CPU。
- **资源生命周期**: 析构函数中调用 `free()` 确保资源释放，`set_buffer()` 在注册新句柄前先释放旧资源。
- **OpenGL 互操作当前被禁用**: 注意在 `HIPDevice::should_use_graphics_interop()` 中，OpenGL 互操作因驱动 bug（AMD 21.40 版本）始终返回 `false`，因此该类在实际运行中暂未被使用。

## 关联文件
- `src/device/hip/queue.h` / `queue.cpp` — 通过 `graphics_interop_create()` 创建本类实例
- `src/device/hip/device_impl.h` / `device_impl.cpp` — `should_use_graphics_interop()` 控制是否启用互操作
- `src/device/graphics_interop.h` — 基类定义
- `src/session/display_driver.h` — 显示驱动与图形互操作缓冲区定义
