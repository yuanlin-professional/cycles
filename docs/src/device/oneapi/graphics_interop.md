# graphics_interop.h / graphics_interop.cpp - oneAPI 图形互操作（Vulkan 内存共享）

## 概述

本文件实现了 oneAPI 设备与 Vulkan 图形 API 之间的内存互操作功能，允许 Cycles 渲染器直接在 GPU 上与 Vulkan 显示管线共享渲染缓冲区，避免不必要的主机-设备间数据拷贝。该实现基于 SYCL `ext::oneapi::experimental` 扩展的外部内存导入/映射 API。整个文件受 `WITH_ONEAPI` 和 `SYCL_LINEAR_MEMORY_INTEROP_AVAILABLE` 双重编译宏保护。

## 类与结构体

### `OneapiDeviceGraphicsInterop`
- **继承**: `DeviceGraphicsInterop`（来自 `device/graphics_interop.h`）
- **功能**: 封装 oneAPI/SYCL 与 Vulkan 之间的外部内存互操作，管理 Vulkan 缓冲区在 SYCL 设备上的导入、映射和释放
- **关键成员**:
  - `queue_` (`OneapiDeviceQueue*`) — 关联的 oneAPI 设备队列
  - `device_` (`OneapiDevice*`) — 关联的 oneAPI 设备（从队列中获取）
  - `buffer_size_` (`size_t`) — 导入缓冲区的字节大小
  - `need_zero_` (`bool`) — 标记缓冲区在下次映射时是否需要清零
  - `sycl_external_memory_` (`sycl::ext::oneapi::experimental::external_mem`) — SYCL 外部内存句柄
  - `sycl_memory_ptr_` (`void*`) — 映射后的设备可访问指针
  - `vulkan_windows_handle_` (`void*`) — Windows 平台下持有的 Vulkan Win32 NT 句柄（需手动关闭）
- **关键方法**:
  - `set_buffer()` — 导入 Vulkan 外部缓冲区
  - `map()` — 返回设备指针并可选清零
  - `unmap()` — 空操作（持久映射）
  - `free()` — 释放所有 SYCL 和平台资源
- **拷贝/移动语义**: 显式删除所有拷贝和移动构造/赋值操作符，确保资源的唯一所有权

## 核心函数

### `OneapiDeviceGraphicsInterop(queue)` (构造函数)
- 保存队列指针，并通过 `static_cast` 从队列的 `device` 成员获取 `OneapiDevice*`

### `~OneapiDeviceGraphicsInterop()` (析构函数)
- 调用 `free()` 释放所有已分配的互操作资源

### `set_buffer(interop_buffer)`
- **空缓冲区处理**: 若 `interop_buffer.is_empty()`，调用 `free()` 释放现有资源并返回
- **零初始化标记**: 通过 `interop_buffer.take_zero()` 累积（OR）到 `need_zero_`
- **无新句柄**: 若 `!interop_buffer.has_new_handle()`，仅更新零初始化标记后返回
- **图形 API 验证**: 仅支持 `GraphicsInteropDevice::VULKAN`，其他类型记录错误并返回
- **外部内存导入**（平台差异）:
  - **Windows**: 使用 `win32_nt_handle` 类型的外部内存描述符，`import_external_memory` 不接管句柄所有权，需自行保存以便后续关闭
  - **Linux**: 使用 `opaque_fd` 类型的外部内存描述符，`import_external_memory` 接管文件描述符所有权
- **线性内存映射**: 调用 `map_external_linear_memory` 将导入的外部内存映射为设备可访问的线性地址空间指针（持久映射，类似 CUDA/HIP 后端的做法）
- **错误恢复**: 导入或映射失败时，释放已获取的资源（SYCL 外部内存、Windows 句柄），记录错误日志

### `map()`
- 若存在有效指针且 `need_zero_` 为 `true`，使用 `sycl_queue->memset` 异步清零缓冲区（与 CUDA 的 `cuMemsetD8Async` 行为一致）
- 返回 `sycl_memory_ptr_` 转换为 `device_ptr`
- 清零失败时返回空指针 `device_ptr(0)`

### `unmap()`
- 空操作。由于采用持久映射策略，缓冲区在 `set_buffer()` 中映射后一直保持映射状态，直到 `free()` 时才取消映射。

### `free()`
- 若 `sycl_external_memory_.raw_handle` 有效:
  1. 调用 `unmap_external_linear_memory` 取消线性内存映射
  2. 调用 `release_external_memory` 释放外部内存句柄
  3. 每步操作均独立 try-catch，确保尽可能多地释放资源
- **Windows 特有**: 调用 `CloseHandle` 关闭 Vulkan NT 句柄
- 重置所有成员到默认状态

## 依赖关系

### 内部头文件
- `device/oneapi/graphics_interop.h` — 本文件自身的头文件声明
- `device/graphics_interop.h` — 基类 `DeviceGraphicsInterop` 定义
- `device/oneapi/device.h` — `OneapiDevice` 前向声明
- `device/oneapi/device_impl.h` — `OneapiDevice` 完整定义（实现文件中引用）
- `device/oneapi/queue.h` — `OneapiDeviceQueue` 定义
- `session/display_driver.h` — `GraphicsInteropBuffer`、`GraphicsInteropDevice` 定义
- `util/windows.h` — Windows API 头文件（条件编译）

### 被引用
- `src/device/oneapi/queue.cpp` — `OneapiDeviceQueue::graphics_interop_create()` 中创建 `OneapiDeviceGraphicsInterop` 实例

## 实现细节 / 关键算法

1. **持久映射策略**: 与 CUDA/HIP 后端保持一致，缓冲区在 `set_buffer()` 中映射后持续有效直到释放。`map()` 和 `unmap()` 仅用于获取指针和可选的清零操作，不执行实际的映射/取消映射。这降低了每帧的映射开销。

2. **平台句柄生命周期差异**:
   - **Windows**: SYCL `import_external_memory` 不接管 Win32 NT 句柄的所有权，因此 `vulkan_windows_handle_` 需在 `free()` 中显式调用 `CloseHandle` 释放。
   - **Linux**: SYCL `import_external_memory` 接管文件描述符的所有权，`free()` 中无需额外关闭。但导入失败时需手动调用 `close()` 关闭文件描述符。

3. **渐进式错误恢复**: `free()` 中 `unmap` 和 `release` 使用独立的 try-catch 块，确保即使取消映射失败，仍尝试释放外部内存句柄。所有错误均通过 `LOG_ERROR` 记录。

4. **异步清零**: `map()` 中的 `memset` 不等待完成事件（与 CUDA 的 `cuMemsetD8Async` 一致），清零操作与后续渲染内核执行可能重叠。

## 关联文件
- `src/device/oneapi/queue.h` / `queue.cpp` — 队列中创建互操作实例
- `src/device/oneapi/device_impl.h` / `device_impl.cpp` — 提供 `sycl_queue()` 和 `should_use_graphics_interop()` 方法
- `src/device/graphics_interop.h` — 跨后端的图形互操作基类
- `src/session/display_driver.h` — Vulkan 显示驱动和缓冲区定义
