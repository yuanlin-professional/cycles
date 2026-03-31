# bvh.h / bvh.mm - Metal 加速结构（层次包围体）的构建与管理

## 概述

本文件实现了 Cycles 渲染器在 Apple Metal 后端下的层次包围体（BVH）加速结构。`BVHMetal` 类继承自通用 `BVH` 基类，利用 Metal 的硬件加速光线追踪 API (`MTLAccelerationStructure`) 构建底层加速结构（BLAS）和顶层加速结构（TLAS），支持三角形网格、毛发曲线、点云三种几何体类型，并支持运动模糊的多关键帧加速结构。

## 类与结构体

### BVHMetal
- **继承**: `BVH`
- **功能**: 封装 Metal 加速结构的完整生命周期管理，包括 BLAS/TLAS 的构建、重拟合（refit）和压缩
- **关键成员**:
  - `accel_struct` (`id<MTLAccelerationStructure>`) — 顶层加速结构实例
  - `null_BLAS` (`id<MTLAccelerationStructure>`) — 空的底层加速结构，用于填充实例数组中的空位
  - `blas_array` (`vector<id<MTLAccelerationStructure>>`) — 全部 BLAS 引用列表（含重复项，与实例一一对应）
  - `unique_blas_array` (`vector<id<MTLAccelerationStructure>>`) — 去重后的唯一 BLAS 列表
  - `device` (`Device*`) — 所属设备指针
  - `motion_blur` (`bool`) — 是否启用运动模糊
  - `use_pcmi` (`bool`) — 是否使用 macOS 15 的逐分量运动插值（Per-Component Motion Interpolation）
  - `extended_limits` (`bool`) — 是否启用扩展限制（用于超大场景）
- **关键方法**:
  - `build(progress, device, queue, refit)` — 完整构建流程入口，先构建 BLAS 再构建 TLAS
  - `build_BLAS(progress, device, queue, refit)` — 根据几何体类型分发到对应的 BLAS 构建方法
  - `build_BLAS_mesh(...)` — 构建三角形网格的 BLAS，支持运动模糊多关键帧
  - `build_BLAS_hair(...)` — 构建毛发曲线的 BLAS，使用 Metal 原生曲线几何描述符（需 macOS 14+）
  - `build_BLAS_pointcloud(...)` — 构建点云的 BLAS，使用 AABB 包围盒描述符
  - `build_TLAS(progress, device, queue, refit)` — 构建顶层加速结构，管理实例变换矩阵
  - `set_accel_struct(new_accel_struct)` — 安全替换加速结构并更新内存统计

### BVHMetalBuildThrottler（内部结构体，定义在 bvh.mm）
- **功能**: 限制并发 BVH 构建数量，防止 GPU 工作集超出安全上限
- **关键成员**:
  - `wired_memory` — 当前已占用的显存资源量
  - `safe_wired_limit` — 安全显存占用上限（为推荐最大工作集的 1/4）
- **关键方法**:
  - `acquire(bytes)` — 阻塞等待直到有足够资源可用
  - `release(bytes)` — 释放已完成构建所占用的资源额度
  - `wait_for_all()` — 等待所有在途构建完成

## 核心函数

- `support_refit_blas()` — 检测当前 macOS 版本是否支持 BLAS 重拟合（macOS 15.2/15.3 有已知 bug，已在 15.4 修复）
- `decomposed_to_component_transform()` — 将 `DecomposedTransform` 转换为 `MTLComponentTransform`，用于 PCMI 运动实例

## 依赖关系

- **内部头文件**:
  - `bvh/bvh.h` — BVH 基类
  - `bvh/params.h` — BVH 构建参数
  - `device/memory.h` — 设备内存管理
  - `device/metal/util.h` — Metal 工具函数
  - `scene/hair.h`, `scene/mesh.h`, `scene/object.h`, `scene/pointcloud.h` — 场景几何体类型
  - `util/progress.h` — 进度报告
- **被引用**:
  - `device/metal/device_impl.h` — MetalDevice 引用 BVHMetal
  - `bvh/metal.mm` — BVH 工厂函数引用本文件

## 实现细节 / 关键算法

1. **BLAS 构建流程**: 将几何数据上传至 `MTLBuffer`，创建对应的几何描述符（三角形/曲线/AABB），调用 `MTLAccelerationStructureCommandEncoder` 进行 GPU 端构建。静态场景下会执行压缩步骤以减小显存占用，动态场景则使用 `PreferFastBuild` + `Refit` 策略。

2. **TLAS 构建**: 遍历场景中的所有对象实例，为每个实例创建 `MTLAccelerationStructureInstanceDescriptor`（或支持运动的变体），设置变换矩阵和对应的 BLAS 引用，然后组装成顶层实例加速结构。

3. **并发构建限流**: 通过全局单例 `g_bvh_build_throttler` 控制 GPU 端 BVH 构建的并发量，避免瞬间消耗过多显存。使用 `acquire/release` 模式实现信号量式的资源管控。

4. **运动模糊支持**: 对于含运动关键帧的几何体，使用 Metal 的 Motion 变体描述符（如 `MTLAccelerationStructureMotionTriangleGeometryDescriptor`），在 macOS 15+ 上还支持 PCMI 以提升运动插值质量。

5. **异步压缩**: BLAS 构建完成后，在异步 `completedHandler` 回调中查询压缩后大小，再提交异步压缩命令，以流水线方式与其他 BLAS 构建并行执行。

## 关联文件

- `src/device/metal/device_impl.h` / `device_impl.mm` — MetalDevice 调用 `build_bvh()` 触发本文件逻辑
- `src/bvh/metal.mm` — BVH 工厂，负责创建 `BVHMetal` 实例
- `src/device/metal/util.h` — 提供 `metal_gpuResourceID()` 等工具函数
