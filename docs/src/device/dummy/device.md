# device.h / device.cpp - 虚拟占位设备实现

## 概述

本文件实现了 Cycles 渲染器的虚拟占位设备（DummyDevice）。当系统无法成功创建真正的渲染设备（如 GPU 设备初始化失败）时，会回退创建此虚拟设备作为占位符。该设备不执行任何实际的内存分配或渲染操作，仅用于携带错误信息并防止空指针崩溃。

## 类与结构体

### DummyDevice
- **继承**: `Device`（定义于 `device/device.h`）
- **功能**: 作为设备创建失败时的安全回退占位对象。所有内存操作和内核功能均为空实现，不执行任何实际工作。
- **关键成员**: `error_msg`（继承自 `Device`）— 构造时从 `DeviceInfo::error_msg` 复制错误信息，用于向上层报告设备创建失败的原因。
- **关键方法**:
  - `get_bvh_layout_mask()` — 返回 `0`，表示不支持任何 BVH 加速结构布局。
  - `mem_alloc()` — 空实现，不分配任何设备内存。
  - `mem_copy_to()` — 空实现，不执行主机到设备的内存拷贝。
  - `mem_copy_from()` — 空实现，不执行设备到主机的内存拷贝。
  - `mem_move_to_host()` — 空实现，不执行内存迁移。
  - `mem_zero()` — 空实现，不执行内存清零。
  - `mem_free()` — 空实现，不释放任何内存。
  - `const_copy_to()` — 空实现，不执行常量内存拷贝。

## 核心函数

### `device_dummy_create()`
```cpp
unique_ptr<Device> device_dummy_create(const DeviceInfo &info,
                                       Stats &stats,
                                       Profiler &profiler,
                                       bool headless);
```
- **功能**: 工厂函数，创建并返回一个 `DummyDevice` 实例的智能指针。
- **调用时机**: 在 `Device::create()` 中，当所有真正的设备后端（CPU、CUDA、OptiX、HIP、Metal、oneAPI）均创建失败（返回 `nullptr`）时被调用，确保始终返回一个有效的设备对象。
- **参数说明**:
  - `info` — 设备信息，其中 `error_msg` 字段记录了设备创建失败的原因。
  - `stats` — 统计信息对象引用。
  - `profiler` — 性能分析器引用。
  - `headless` — 是否为无头模式（无显示输出）。

## 依赖关系

- **内部头文件**:
  - `device/dummy/device.h` — 本模块头文件，声明 `device_dummy_create()` 工厂函数。
  - `device/device.h` — 基类 `Device` 和 `DeviceInfo` 的定义。
  - `device/queue.h` — 设备队列相关定义（虽然 DummyDevice 未使用队列功能，但包含了此头文件）。
  - `util/unique_ptr.h` — 智能指针 `unique_ptr` 和 `make_unique` 的封装。
- **被引用**:
  - `src/device/device.cpp` — 在 `Device::create()` 工厂方法中，作为所有设备后端创建失败时的最终回退方案调用 `device_dummy_create()`。

## 实现细节

DummyDevice 的设计遵循空对象模式（Null Object Pattern）：

1. **构造函数**: 将 `DeviceInfo` 中的 `error_msg` 复制到自身的 `error_msg` 成员。上层代码可通过 `Device::have_error()` 检测到设备处于错误状态，并通过 `Device::error_message()` 获取具体错误原因。

2. **BVH 支持**: `get_bvh_layout_mask()` 返回 `0`（即 `BVH_LAYOUT_NONE`），明确表示该设备不支持任何类型的 BVH 加速结构。

3. **内存操作**: 所有内存操作方法（`mem_alloc`、`mem_copy_to`、`mem_copy_from`、`mem_move_to_host`、`mem_zero`、`mem_free`、`const_copy_to`）均为空函数体，不执行任何操作。这保证了即使上层代码在错误检查前意外调用了内存操作，也不会导致崩溃。

4. **析构函数**: 使用 `= default`，无需额外清理资源。

## 关联文件

- `src/device/device.h` — `Device` 基类定义，包含所有虚函数接口声明。
- `src/device/device.cpp` — 设备工厂方法 `Device::create()` 和 `Device::dummy_device()` 静态方法。
- `src/device/cpu/device.cpp` — CPU 设备实现，与 DummyDevice 同为非 GPU 设备。
- `src/device/memory.h` — `device_memory` 类定义，DummyDevice 所有内存操作参数的类型。
