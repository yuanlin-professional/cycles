# debug.h / debug.cpp - 调试标志注册表

## 概述

`debug.h` 和 `debug.cpp` 实现了 Cycles 的全局调试标志注册表 `DebugFlags`，用于在开发阶段微调各渲染后端（CPU、CUDA、HIP、OptiX、Metal）的行为。这些标志可通过环境变量控制，不对外暴露为正式用户设置。`DebugFlags` 采用单例模式，提供便捷的访问函数 `DebugFlags()`。

## 类与结构体

### `DebugFlags`

全局调试标志注册表（单例），包含各后端的子结构体。

| 方法 | 说明 |
|------|------|
| `static DebugFlags &get()` | 获取单例实例 |
| `void reset()` | 重置所有后端标志为默认值 |

**单例实现**：私有默认构造函数，删除拷贝构造和赋值运算符。

#### `DebugFlags::CPU`

| 字段 | 默认值 | 环境变量 | 说明 |
|------|--------|----------|------|
| `bool avx2` | `true` | `CYCLES_CPU_NO_AVX2` | 是否允许使用 AVX2 指令集 |
| `bool sse42` | `true` | — | 是否允许使用 SSE4.2 指令集 |
| `BVHLayout bvh_layout` | `BVH_LAYOUT_AUTO` | — | 请求的 BVH 布局（默认自动选择最快） |

| 方法 | 说明 |
|------|------|
| `bool has_avx2()` | 检测 AVX2 是否可用（需 SSE4.2 也启用） |
| `bool has_sse42()` | 检测 SSE4.2 是否可用 |
| `void reset()` | 从环境变量重新读取标志 |

#### `DebugFlags::CUDA`

| 字段 | 默认值 | 环境变量 | 说明 |
|------|--------|----------|------|
| `bool adaptive_compile` | `false` | `CYCLES_CUDA_ADAPTIVE_COMPILE` | 是否启用基于特征的自适应运行时编译 |

#### `DebugFlags::HIP`

| 字段 | 默认值 | 环境变量 | 说明 |
|------|--------|----------|------|
| `bool adaptive_compile` | `false` | `CYCLES_HIP_ADAPTIVE_COMPILE` | 是否启用自适应运行时编译 |

#### `DebugFlags::OptiX`

| 字段 | 默认值 | 说明 |
|------|--------|------|
| `bool use_debug` | `false` | 启用调试模式（降低优化级别、启用验证、提高日志详细度） |

#### `DebugFlags::Metal`

| 字段 | 默认值 | 环境变量 | 说明 |
|------|--------|----------|------|
| `bool adaptive_compile` | `false` | `CYCLES_METAL_ADAPTIVE_COMPILE` | 自适应运行时编译 |
| `bool use_local_atomic_sort` | `true` | `CYCLES_METAL_LOCAL_ATOMIC_SORT` | 本地原子排序 |
| `bool use_nanovdb` | `true` | `CYCLES_METAL_NANOVDB` | NanoVDB 支持 |
| `bool use_async_pso_creation` | `true` | `CYCLES_METAL_ASYNC_PSO_CREATION` | 异步管线状态对象创建 |
| `bool use_metalrt_pcmi` | `true` | `CYCLES_METALRT_PCMI` | 按分量运动插值 |

## 核心函数/宏定义

### 类型别名

| 别名 | 说明 |
|------|------|
| `DebugFlagsRef` | `DebugFlags &` |
| `DebugFlagsConstRef` | `const DebugFlags &` |

### 便捷函数

```cpp
inline DebugFlags &DebugFlags()
```

返回 `DebugFlags::get()` 的引用，允许以函数调用方式访问单例（与类名同名函数）。

## 依赖关系

- **内部头文件**: `bvh/params.h`（提供 `BVHLayout` 枚举）, `util/log.h`（日志输出）
- **外部依赖**: `<cassert>`, `<cstdlib>`（`getenv`）
- **被引用**: `util/debug.cpp`, `device/metal/device.mm`, `device/cpu/kernel_function.h`

## 实现细节

1. **环境变量驱动**：各后端 `reset()` 函数通过 `getenv()` 读取特定环境变量来禁用/启用功能。例如设置 `CYCLES_CPU_NO_AVX2` 环境变量（任意值）将禁用 AVX2 指令集。

2. **层级检测**：CPU 指令集检测采用层级模式——`has_avx2()` 依赖 `has_sse42()` 返回 true，确保低级指令集也被启用。

3. **Metal 多标志**：Metal 后端有最多的环境变量控制，反映了 Apple GPU 后端的特殊优化选项。使用 `atoi` 将环境变量值转换为布尔值（0 = false）。

4. **BVH 布局调试**：`bvh_layout` 允许开发者强制使用特定 BVH 布局（如 BVH2、BVH4），用于调试其他 CPU/GPU 使用的 BVH 变体。

## 关联文件

- `bvh/params.h` - 定义 `BVHLayout` 枚举
- `device/cpu/kernel_function.h` - 根据调试标志选择 CPU 内核
- `device/metal/device.mm` - 读取 Metal 调试标志
