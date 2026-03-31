# optix.h / optix.cpp - OptiX 光线追踪加速结构封装

## 概述

本模块提供了 NVIDIA OptiX 光线追踪引擎的层次包围体(BVH)封装类 `BVHOptiX`。它继承自通用的 `BVH` 基类，管理 OptiX 专用的加速结构（Acceleration Structure）数据，包括遍历句柄、加速结构数据缓冲区和运动变换数据。整个模块在 `WITH_OPTIX` 宏保护下，仅在支持 OptiX 的构建配置中编译。

## 类与结构体

### `BVHOptiX`
继承自 `BVH` 基类，是 OptiX 加速结构的具体实现类。

| 成员 | 类型 | 说明 |
|------|------|------|
| `device` | `Device *` | 指向创建此 BVH 的设备 |
| `traversable_handle` | `uint64_t` | OptiX 遍历句柄，用于光线遍历加速结构 |
| `as_data` | `unique_ptr<device_only_memory<char>>` | 加速结构数据的设备端内存（TLAS 或 BLAS） |
| `motion_transform_data` | `unique_ptr<device_only_memory<char>>` | 运动变换数据的设备端内存 |

**构造函数:**
```cpp
BVHOptiX(const BVHParams &params,
         const vector<Geometry *> &geometry,
         const vector<Object *> &objects,
         Device *device);
```
- 调用基类 `BVH` 构造函数
- 初始化 `traversable_handle` 为 0
- 分配设备端内存缓冲区，根据 `params.top_level` 区分命名为 "optix tlas"（顶层加速结构）或 "optix blas"（底层加速结构）

**析构函数:**
- 调用 `device->release_bvh(this)` 进行延迟释放
- 加速结构内存的释放被延迟处理，因为删除 BVH 可能发生在渲染仍在使用该结构时

## 核心函数

### `BVHOptiX::BVHOptiX(...)`
构造函数初始化列表中：
- `as_data` 根据是否为顶层 BVH 使用不同的调试名称（"optix tlas" vs "optix blas"）
- `motion_transform_data` 用于存储运动模糊的变换数据
- 均使用 `device_only_memory` 模板，表示这些内存仅存在于设备端（GPU）

### `BVHOptiX::~BVHOptiX()`
析构时通过 `device->release_bvh(this)` 通知设备释放加速结构。这里使用延迟释放机制，因为 BVH 的销毁可能发生在渲染管线仍在访问该加速结构的时刻，直接释放会导致竞态条件。

## 依赖关系

- **内部头文件**:
  - `bvh/bvh.h` - BVH 基类定义
  - `bvh/params.h` - BVH 参数（`BVHParams`）
  - `device/memory.h` - 设备内存管理（`device_only_memory`）
  - `device/device.h` - 设备接口（`Device`）
  - `util/unique_ptr.h` - 智能指针工具
- **被引用**: `bvh/bvh.cpp`、`device/optix/device_impl.cpp`

## 实现细节 / 关键算法

### TLAS 与 BLAS 的区分
OptiX 使用两级加速结构：
- **BLAS（Bottom-Level Acceleration Structure）**: 底层加速结构，对应单个几何体的三角形/曲线等图元
- **TLAS（Top-Level Acceleration Structure）**: 顶层加速结构，对应整个场景的对象实例

通过 `params.top_level` 标志区分，构造时给设备内存赋予不同的调试名称。

### 延迟内存释放
BVH 对象的析构不直接释放 GPU 内存，而是调用 `device->release_bvh(this)` 委托设备层管理。这是因为 GPU 渲染是异步的，BVH 被标记为删除时，之前提交的渲染命令可能仍在使用该加速结构的 GPU 内存。

### 条件编译
整个模块由 `#ifdef WITH_OPTIX` 包裹，只有在编译时启用 OptiX 支持才会包含此代码。实际的加速结构构建逻辑不在此文件中，而是在 `device/optix/device_impl.cpp` 中通过 OptiX API 完成。

## 关联文件

- `bvh/bvh.h` / `bvh/bvh.cpp` - BVH 基类，提供通用 BVH 接口
- `bvh/params.h` - BVH 参数配置
- `device/optix/device_impl.cpp` - OptiX 设备实现，实际调用 OptiX API 构建加速结构
- `device/memory.h` - `device_only_memory` 设备端内存模板
- `device/device.h` - `Device::release_bvh()` 延迟释放接口
