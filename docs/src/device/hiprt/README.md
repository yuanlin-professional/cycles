# device/hiprt - AMD HIPRT 硬件光线追踪设备实现

## 概述

`hiprt/` 子目录实现了 Cycles 的 AMD HIPRT 硬件光线追踪后端。`HIPRTDevice` 继承自 `HIPDevice`，在 HIP 后端的基础上集成了 AMD HIPRT（HIP Ray Tracing）库，利用 RDNA 2 及更新 GPU 的硬件光线追踪单元加速光线-场景求交。

核心特点：
- **硬件加速 BVH**：使用 HIPRT 构建 BLAS（底层加速结构）和 TLAS（顶层加速结构），替代软件 BVH。
- **自定义交叉函数**：通过 `hiprtFuncTable` 注册自定义交叉函数，支持曲线、点云、运动模糊三角形等非标准图元。
- **全局栈缓冲区**：使用 `hiprtGlobalStackBuffer` 为光线遍历提供全局栈内存。
- **运动模糊支持**：通过多变换矩阵和 `hiprtTransformHeader` 实现实例级运动模糊。

## 目录结构

| 文件名 | 类型 | 说明 |
|--------|------|------|
| `device_impl.h` | 头文件 | `HIPRTDevice` 类定义 |
| `device_impl.cpp` | 源文件 | `HIPRTDevice` 实现：BLAS/TLAS 构建、内核编译、交叉函数表管理 |
| `queue.h` | 头文件 | `HIPRTDeviceQueue` 命令队列类 |
| `queue.cpp` | 源文件 | 扩展 `HIPDeviceQueue` 的 `enqueue()` 以注入 HIPRT 全局栈参数 |

## 核心类与数据结构

### HIPRTDevice

定义位置：`device_impl.h`

继承自 `HIPDevice`。关键成员：

| 成员 | 说明 |
|------|------|
| `hiprt_context` | HIPRT 上下文 (`hiprtContext`) |
| `scene` | HIPRT 场景（TLAS） |
| `functions_table` | 交叉函数表 (`hiprtFuncTable`)，注册自定义图元的交叉/过滤函数 |
| `global_stack_buffer` | 光线遍历全局栈缓冲区 (`hiprtGlobalStackBuffer`) |
| `scratch_buffer` | BVH 构建临时缓冲区 |
| `stale_bvh` | 延迟释放的旧 BVH 几何体列表 |
| `prim_visibility` | 每个对象的可见性标记 |
| `instance_transform_matrix` | 实例变换矩阵（用于 TLAS 构建） |
| `transform_headers` | 变换头信息（运动模糊时映射实例到多个变换矩阵） |
| `user_instance_id` | HIPRT 实例 ID 到应用层实例 ID 的映射 |
| `hiprt_blas_ptr` / `blas_ptr` | BLAS 指针列表 |
| `custom_prim_info` / `custom_prim_info_offset` | 自定义图元信息与偏移 |
| `prims_time` / `prim_time_offset` | 运动模糊图元时间信息 |

关键方法：

| 方法 | 说明 |
|------|------|
| `build_bvh()` | 构建 HIPRT 加速结构（BLAS + TLAS） |
| `release_bvh()` | 释放加速结构资源 |
| `prepare_triangle_blas()` | 为三角形网格准备 BLAS 构建输入 |
| `prepare_curve_blas()` | 为曲线（毛发）准备 BLAS 构建输入 |
| `prepare_point_blas()` | 为点云准备 BLAS 构建输入 |
| `build_blas()` | 构建单个几何体的 BLAS |
| `build_tlas()` | 构建场景级 TLAS |
| `compile_kernel()` | 编译 HIPRT 内核（使用 `hiprt` 基目录） |
| `const_copy_to()` | 扩展父类方法，同步 HIPRT 特有的全局参数 |
| `load_kernels()` | 加载 HIPRT 内核并创建交叉函数表 |

### HIPRTDeviceQueue

定义位置：`queue.h`

继承自 `HIPDeviceQueue`。扩展 `enqueue()` 方法，在内核启动时自动注入 HIPRT 全局栈缓冲区参数（`HIPRT_GLOBAL_STACK` 类型参数）。

### Filter_Function 枚举

自定义交叉过滤函数类型：
- `Closest` — 最近相交
- `Shadows` — 阴影射线
- `Local` — 局部相交（次表面散射）
- `Volume` — 体积相交

### Primitive_Type 枚举

HIPRT 支持的图元类型：
- `Triangle` — 三角形
- `Curve` — 曲线（毛发）
- `Motion_Triangle` — 运动模糊三角形
- `Point` — 点云

## 硬件要求

- **GPU**：AMD RDNA 2 及更新架构（gfx1030 及以上）
- **驱动**：支持 HIPRT 的 AMD 驱动
- **HIPRT SDK**：编译时需要 HIPRT SDK；运行时通过 `hiprtew`（HIPRT Extension Wrangler）动态加载
- **条件编译**：`WITH_HIPRT` 和 `WITH_HIP`

## API 封装

- **HIPRT API**：通过 `hiprtew.h` 动态加载或直接链接 `hiprt/hiprt_types.h`
- 核心 API：`hiprtCreateContext`、`hiprtCreateGeometry`、`hiprtBuildGeometry`、`hiprtCreateScene`、`hiprtBuildScene`、`hiprtCreateFuncTable`
- BVH 构建：`hiprtGeometryBuildInput`、`hiprtSceneBuildInput`、`hiprtBuildOptions`
- 全局栈：`hiprtGlobalStackBuffer`

## 依赖关系

### 上游依赖（本模块依赖）

| 模块 | 用途 |
|------|------|
| `device/hip/` | `HIPDevice` 基类、`HIPDeviceQueue` 基类 |
| `device/` | 设备抽象、内存体系 |
| `bvh/` | BVH 参数、BVHHIPRT 数据结构 |
| `scene/` | `Mesh`、`Hair`、`PointCloud`、`Object` 等几何数据 |
| HIPRT SDK / HIPRTEW | HIPRT 光线追踪 API |

### 下游依赖（依赖本模块）

| 模块 | 用途 |
|------|------|
| `device/device.cpp` | 工厂方法（通过 HIP 后端路径创建 HIPRT 设备） |

## 参见

- `src/device/hip/` — HIP 后端基类
- `src/device/optix/` — 类似的 NVIDIA 硬件光线追踪实现
- `src/bvh/` — BVH 加速结构
- `src/kernel/device/hip/` — HIP/HIPRT 内核实现
