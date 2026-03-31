# device/dummy - 空设备（错误回退）

## 概述

`dummy/` 子目录实现了 Cycles 的空设备后端。`DummyDevice` 继承自 `Device` 基类，所有方法均为空操作（no-op）。当用户选择的渲染设备创建失败（例如 GPU 驱动不可用、CUDA/OptiX 初始化失败）时，`Device::create()` 工厂方法会回退创建一个 `DummyDevice`，用于安全地传递错误信息而不导致崩溃。

## 目录结构

| 文件名 | 类型 | 说明 |
|--------|------|------|
| `device.h` | 头文件 | `device_dummy_create()` 工厂函数 |
| `device.cpp` | 源文件 | `DummyDevice` 类定义与实现 |

## 核心类与数据结构

### DummyDevice

定义位置：`device.cpp`

继承自 `Device`。特点：
- 构造函数从 `DeviceInfo::error_msg` 获取错误信息，存入 `error_msg` 成员。
- `get_bvh_layout_mask()` 返回 0（不支持任何 BVH 布局）。
- 所有内存操作（`mem_alloc`、`mem_copy_to`、`mem_move_to_host`、`mem_copy_from`、`mem_zero`、`mem_free`、`const_copy_to`）均为空操作。
- 不支持内核加载、命令队列创建或渲染。

### 使用场景

`DummyDevice` 在以下情况被创建：
1. 用户选择的 GPU 类型（CUDA/OptiX/HIP/Metal/oneAPI）初始化失败。
2. `Device::create()` 中 `switch` 没有匹配到任何有效设备类型。
3. 通过 `Device::dummy_device()` 静态方法创建带有自定义错误信息的 `DeviceInfo`。

上层代码（`Session`）在检测到 `Device::have_error()` 返回 true 时，会向用户报告错误并中止渲染。

## 硬件要求

无任何硬件要求。`DummyDevice` 可在任何平台上创建。

## API 封装

不封装任何外部 API。

## 依赖关系

### 上游依赖（本模块依赖）

| 模块 | 用途 |
|------|------|
| `device/` | `Device` 基类 |
| `device/queue.h` | `DeviceQueue` 类型声明 |

### 下游依赖（依赖本模块）

| 模块 | 用途 |
|------|------|
| `device/device.cpp` | 工厂方法在设备创建失败时回退到 `device_dummy_create()` |

## 参见

- `src/device/device.cpp` — `Device::create()` 工厂方法（回退逻辑）
- `src/device/device.h` — `Device` 基类、`Device::dummy_device()` 静态方法
