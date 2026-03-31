# graphics_interop.h / graphics_interop.mm - Metal 图形互操作接口

## 概述

本文件实现了 Metal 设备与外部图形 API（如 OpenGL/Vulkan 显示驱动）之间的资源共享机制。`MetalDeviceGraphicsInterop` 类允许 Cycles 渲染结果直接写入由显示驱动提供的 Metal 缓冲区，避免在渲染与显示之间进行额外的内存拷贝，从而实现高效的视口渲染。

## 类与结构体

### MetalDeviceGraphicsInterop
- **继承**: `DeviceGraphicsInterop`
- **功能**: 管理 Metal 渲染设备与图形显示系统之间的缓冲区共享
- **关键成员**:
  - `queue_` (`MetalDeviceQueue*`) — 关联的 Metal 命令队列
  - `device_` (`MetalDevice*`) — 关联的 Metal 设备
  - `mem_` (`MetalDevice::MetalMem`) — 封装共享 Metal 缓冲区的内存描述符
  - `size_` (`size_t`) — 当前缓冲区大小
  - `need_zero_` (`bool`) — 标记缓冲区是否需要在下次映射时清零
- **关键方法**:
  - `set_buffer(interop_buffer)` — 接收来自图形 API 的缓冲区句柄，将原生句柄 `reinterpret_cast` 为 `id<MTLBuffer>`
  - `map()` — 映射缓冲区供 Cycles 渲染器写入，若 `need_zero_` 为真则先清零缓冲区内容
  - `unmap()` — 取消映射（Apple Silicon UMA 下为空操作）
- **复制/移动语义**: 显式删除拷贝构造、移动构造、拷贝赋值和移动赋值运算符，确保资源所有权唯一

## 核心函数

本文件的核心逻辑集中在 `MetalDeviceGraphicsInterop` 类的方法中，无独立全局函数。

## 依赖关系

- **内部头文件**:
  - `device/graphics_interop.h` — 图形互操作基类定义
  - `device/metal/device_impl.h` — MetalDevice 和 MetalMem 类型
  - `session/display_driver.h` — 显示驱动接口
- **被引用**:
  - `device/metal/queue.mm` — `MetalDeviceQueue::graphics_interop_create()` 创建本类实例

## 实现细节 / 关键算法

1. **缓冲区句柄转换**: `set_buffer()` 将 `GraphicsInteropBuffer` 中的原生句柄通过 `reinterpret_cast` 转换为 `id<MTLBuffer>`。这要求显示驱动提供的是兼容的 Metal 缓冲区。

2. **延迟清零**: 当显示驱动请求清零目标缓冲区时，`need_zero_` 标志被设置，实际清零操作延迟到 `map()` 调用时通过 `memset` 执行。这是因为在 Apple Silicon 的统一内存架构下，直接 CPU 端 `memset` 是最高效的方式。

3. **空缓冲区处理**: 当 `interop_buffer.is_empty()` 为真时，清除内部缓冲区引用，后续 `map()` 调用将返回空指针。

4. **设备指针编码**: `map()` 返回 `device_ptr(&mem_)`，将 `MetalMem` 指针编码为设备指针，与 `MetalDevice` 的内存管理约定一致。

## 关联文件

- `src/device/metal/queue.h` / `queue.mm` — 创建图形互操作实例的入口
- `src/device/metal/device_impl.h` / `device_impl.mm` — MetalDevice 提供 `should_use_graphics_interop()` 判断
- `src/device/graphics_interop.h` — 基类定义
- `src/session/display_driver.h` — 显示驱动端的缓冲区接口
