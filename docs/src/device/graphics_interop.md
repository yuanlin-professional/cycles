# graphics_interop.h / graphics_interop.cpp - 图形 API 互操作接口

## 概述

本文件定义了设备侧图形互操作的抽象接口 `DeviceGraphicsInterop`，用于支持 Cycles 渲染器将计算结果直接写入 OpenGL、Vulkan 或 Metal 等图形 API 的缓冲区。该接口是一个轻量级抽象基类，具体实现由各 GPU 后端（CUDA、HIP、Metal、oneAPI）提供。当前 `.cpp` 文件为空实现，所有逻辑均在派生类中。

## 类与结构体

### DeviceGraphicsInterop
- **继承**: 无（抽象基类）
- **功能**: 管理设备与图形 API 之间缓冲区互操作所需的全部句柄，提供资源映射/解映射的统一接口。
- **关键成员**: 无数据成员（纯接口类）
- **关键方法**:
  - `set_buffer(GraphicsInteropBuffer &interop_buffer)` — 纯虚方法，使用给定的图形资源信息更新设备侧互操作缓冲区
  - `map()` — 纯虚方法，将图形资源映射为设备可用的 `device_ptr`，返回设备指针
  - `unmap()` — 纯虚方法，解除图形资源的映射

## 核心函数

本文件无独立函数，所有功能通过 `DeviceGraphicsInterop` 的虚方法接口定义。

## 依赖关系

- **内部头文件**: `util/types.h`
- **外部库**: 无直接依赖（具体图形 API 依赖在各后端实现中）
- **被引用**:
  - `src/device/queue.h` — `DeviceQueue` 提供 `graphics_interop_create()` 工厂方法
  - `src/integrator/path_trace_work_gpu.h` — GPU 路径追踪工作类使用互操作

## 实现细节 / 关键算法

- `DeviceGraphicsInterop` 采用经典的纯虚接口模式。`GraphicsInteropBuffer` 前向声明为外部类型，封装了图形 API 侧的缓冲区资源描述信息。
- 互操作的生命周期为：通过 `DeviceQueue::graphics_interop_create()` 创建实例，调用 `set_buffer()` 绑定图形资源，使用 `map()`/`unmap()` 对在内核执行前后映射和释放缓冲区。
- `.cpp` 文件当前仅包含命名空间声明，无实质实现代码——这是因为该类是纯虚基类，所有方法实现在各后端的派生类中完成。

## 关联文件

- `src/device/cuda/graphics_interop.h` / `.cpp` — CUDA 后端图形互操作实现
- `src/device/hip/graphics_interop.h` / `.cpp` — HIP 后端图形互操作实现
- `src/device/metal/graphics_interop.h` / `.mm` — Metal 后端图形互操作实现
- `src/device/oneapi/graphics_interop.h` / `.cpp` — oneAPI 后端图形互操作实现
- `src/device/queue.h` — 通过 `graphics_interop_create()` 工厂方法创建互操作实例
- `src/integrator/path_trace_work_gpu.h` — 在 GPU 路径追踪中使用互操作进行视口显示
