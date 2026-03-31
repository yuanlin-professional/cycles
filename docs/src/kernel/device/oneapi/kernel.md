# kernel.cpp / kernel.h - oneAPI 内核入口、加载与调度

## 概述

本文档覆盖两个文件：`kernel.h`（头文件/公共接口）和 `kernel.cpp`（实现）。它们共同构成 Cycles 渲染器 oneAPI 后端的主内核模块，负责内核编译加载、设备能力测试、内核调度执行等核心功能。

`kernel.h` 声明了通过动态链接库导出的 C 接口函数，供主机端设备管理代码调用。`kernel.cpp` 包含完整的 SYCL 实现，组装所有内核头文件以编译 GPU 内核代码，并提供内核调度的 switch-case 分发逻辑。

## 核心函数/宏定义

### 公共接口（kernel.h）

#### 类型与结构体

| 定义 | 说明 |
|------|------|
| `OneAPIErrorCallback` | 错误回调函数类型：`void (*)(const char *error, void *user_ptr)` |
| `KernelContext` | 内核执行上下文，包含 `SyclQueue*`、`kernel_globals` 指针和 `scene_max_shaders` |
| `CYCLES_KERNEL_ONEAPI_EXPORT` | DLL 导出/导入宏（Windows `__declspec(dllexport/dllimport)`，Linux `__attribute__((visibility("default")))` ） |

#### 导出函数

| 函数 | 说明 |
|------|------|
| `oneapi_run_test_kernel(SyclQueue*)` | 执行简单的测试内核验证设备基本功能 |
| `oneapi_zero_memory_on_device(SyclQueue*, void*, size_t)` | 将设备内存清零 |
| `oneapi_set_error_cb(OneAPIErrorCallback, void*)` | 设置全局错误回调 |
| `oneapi_suggested_gpu_kernel_size(DeviceKernel)` | 返回特定内核的建议工作组大小 |
| `oneapi_enqueue_kernel(KernelContext*, ...)` | 将指定内核提交到 SYCL 队列执行 |
| `oneapi_load_kernels(SyclQueue*, uint, bool)` | 预编译/加载指定特性集的内核 |

### 实现（kernel.cpp）

#### 头文件组装链

```
kernel.h → compat.h → globals.h → kernel_templates.h → gpu/kernel.h → device/kernel.cpp
```

#### Embree 特性标志映射

```cpp
static RTCFeatureFlags oneapi_embree_features_from_kernel_features(const uint kernel_features)
```

将 Cycles 内核特性标志映射到 Embree RTC 特性标志：

| Cycles 特性 | Embree 标志 |
|-------------|------------|
| `KERNEL_FEATURE_HAIR_THICK` | `RTC_FEATURE_FLAG_ROUND_CATMULL_ROM_CURVE`, `RTC_FEATURE_FLAG_ROUND_LINEAR_CURVE` |
| `KERNEL_FEATURE_HAIR` | `RTC_FEATURE_FLAG_FLAT_CATMULL_ROM_CURVE` |
| `KERNEL_FEATURE_POINTCLOUD` | `RTC_FEATURE_FLAG_POINT` |
| `KERNEL_FEATURE_OBJECT_MOTION` | `RTC_FEATURE_FLAG_MOTION_BLUR` |

#### 内核大小建议

`oneapi_suggested_gpu_kernel_size()` 为不同类型的内核返回优化的工作组大小：
- 路径数组操作 → `GPU_PARALLEL_ACTIVE_INDEX_DEFAULT_BLOCK_SIZE`
- 排序操作 → `GPU_PARALLEL_SORTED_INDEX_DEFAULT_BLOCK_SIZE` 或 `GPU_PARALLEL_SORT_BLOCK_SIZE`
- 前缀和 → `GPU_PARALLEL_PREFIX_SUM_DEFAULT_BLOCK_SIZE`
- 其他 → 0（使用默认值）

#### 内核过滤函数

| 函数 | 说明 |
|------|------|
| `oneapi_kernel_is_required_for_features()` | 根据内核特性判断某内核是否需要编译（按名称匹配） |
| `oneapi_kernel_is_compatible_with_hardware_raytracing()` | 检查内核与硬件光线追踪的兼容性（Embree < 4.1 排除 MNEE/Raytrace 内核） |
| `oneapi_kernel_has_intersections()` | 检查内核是否包含求交操作 |

#### 内核加载（oneapi_load_kernels）

分两阶段处理：
1. **Embree GPU 内核**（启用硬件光线追踪时）：JIT 编译带有 Embree 特化常量的求交内核
2. **常规内核**：确保 AoT 或缓存的 JIT 二进制可用

#### 内核调度（oneapi_enqueue_kernel）

通过大型 `switch` 语句分发所有设备内核，使用 `oneapi_call()` 模板函数将 `void**` 参数数组自动转型为正确的类型。覆盖的内核类别包括：

- **积分器内核**：初始化、求交、着色（表面/体积/阴影/背景/光源）
- **路径管理内核**：排队、排序、压缩、活跃/终止路径
- **自适应采样内核**：收敛检查和滤波
- **胶片转换内核**：各种输出格式（深度、雾、运动、密码遮罩等）
- **着色器评估内核**：位移、背景、曲线阴影透明度、体积密度
- **辅助内核**：前缀和、去噪预处理、加密遮罩后处理

特殊处理：排序内核（`SORT_BUCKET_PASS`、`SORT_WRITE_PASS`）需要额外的 `sycl::local_accessor<int>` 共享内存，使用 `scene_max_shaders` 确定大小。

#### 测试内核

`oneapi_run_test_kernel()` 执行简单的"数组加法"验证：
1. 在主机分配数组 A，填充随机数
2. 复制到设备
3. 执行 `B[i] = A[i] + i` 内核
4. 复制回主机并验证结果

## 依赖关系

### kernel.h

- **内部头文件**: `<stddef.h>`
- **被引用**:
  - `device/oneapi/queue.h` — 设备队列管理
  - `device/oneapi/queue.cpp` — 内核提交实现
  - `device/oneapi/device_impl.h` — 设备实现
  - `device/oneapi/device_impl.cpp` — 设备初始化

### kernel.cpp

- **内部头文件**:
  - `kernel.h` — 公共接口
  - `kernel/device/oneapi/compat.h` — oneAPI 兼容层
  - `kernel/device/oneapi/globals.h` — 全局数据结构
  - `kernel/device/oneapi/kernel_templates.h` — 调用模板
  - `kernel/device/gpu/kernel.h` — GPU 通用内核代码
  - `device/kernel.cpp` — 设备内核辅助函数
  - `<sycl/sycl.hpp>` — SYCL 运行时

## 实现细节 / 关键算法

### 动态链接库架构

oneAPI 内核被编译为独立的动态链接库（Windows 上为 DLL），通过 `extern "C"` 导出接口函数。主机端通过动态加载获取函数指针进行调用。这允许 oneAPI 内核独立于主程序编译，支持不同的编译器工具链。

### JIT 编译与特化常量

当启用 Embree GPU 硬件光线追踪时，需要为不同的特性配置 JIT 编译内核：

```
sycl::kernel_bundle<input> → set_specialization_constant<oneapi_embree_features>(flags) → sycl::build()
```

非求交内核和禁用硬件光线追踪的求交内核设置 `RTC_FEATURE_FLAG_NONE` 进行编译。

### 编译器完整性检查

在 `oneapi_enqueue_kernel` 的 `switch` 语句中使用编译器 pragma 将 `-Wswitch` 设为错误级别，确保新增的 `DeviceKernel` 枚举值不会被遗漏。

### 胶片转换宏展开

使用 `DEVICE_KERNEL_FILM_CONVERT` / `DEVICE_KERNEL_FILM_CONVERT_PARTIAL` 宏批量生成所有胶片转换内核的 case 分支，每种格式生成标准版和半精度 RGBA 版两个变体。

## 关联文件

- `kernel/device/oneapi/compat.h` — oneAPI 兼容层
- `kernel/device/oneapi/globals.h` — 全局数据结构
- `kernel/device/oneapi/kernel_templates.h` — `oneapi_call` 调用模板
- `kernel/device/gpu/kernel.h` — GPU 通用内核实现
- `device/oneapi/queue.cpp` — 主机端内核提交
- `device/oneapi/device_impl.cpp` — 主机端设备管理
