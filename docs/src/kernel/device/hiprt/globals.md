# globals.h - HIPRT 设备全局数据结构与内核参数定义

## 概述

本文件定义了 HIPRT 后端的全局常量数据结构，包括 GPU 内核全局状态 `KernelGlobalsGPU`、内核参数结构 `KernelParamsHIPRT`，以及自定义求交与过滤函数的函数表索引枚举。它还定义了 HIPRT 特有的共享内存栈尺寸常量和数据访问抽象宏，是 HIPRT 后端所有内核文件的基础头文件。

## 核心函数/宏定义

### 常量定义

- **`HIPRT_THREAD_STACK_SIZE (64)`** - 每个线程在全局栈缓冲区中预留的栈大小。
- **`HIPRT_SHARED_STACK_SIZE (24)`** - 每个线程的本地数据存储（LDS）栈分配大小，经验值。
- **`HIPRT_THREAD_GROUP_SIZE (256)`** - 求交内核每个工作组的线程数。由于 HIPRT 求交内核使用本地内存且其大小随线程数线性增长，因此从默认的 1024 降低到 256 以避免超出最大本地内存限制。

### 结构体

- **`KernelGlobalsGPU`** - GPU 内核全局状态，包含 `hiprtGlobalStackBuffer`（全局栈缓冲区）和 `hiprtSharedStackBuffer`（共享栈）。用于在遍历过程中维护 BVH 栈状态。
- **`KernelParamsHIPRT`** - HIPRT 内核启动参数，包含：
  - `KernelData data` - 全局场景数据
  - HIPRT 专有数据数组：`user_instance_id`、`blas_ptr`、`custom_prim_info`、`custom_prim_info_offset`、`prims_time`、`prim_time_offset`
  - 通用内核数据数组（通过 `kernel/data_arrays.h` 包含）
  - `IntegratorStateGPU` 积分器状态
  - 四个 `hiprtFuncTable`：`table_closest_intersect`、`table_shadow_intersect`、`table_local_intersect`、`table_volume_intersect`

### 类型别名

- **`KernelGlobals`** - 定义为 `ccl_global KernelGlobalsGPU *ccl_restrict`，指向 GPU 全局状态的指针。
- **`Stack`** - `hiprtGlobalStack` 类型别名。
- **`Instance_Stack`** - `hiprtEmptyInstanceStack` 类型别名。

### 初始化宏

- **`HIPRT_INIT_KERNEL_GLOBAL()`** - 内核入口点初始化宏。分配共享内存数组，创建 `KernelGlobalsGPU` 实例，初始化共享栈数据指针、栈大小和全局栈缓冲区。

### 枚举

- **`Intersection_Function_Table_Index`** - 自定义求交函数表索引。为四种光线类型（closest/shadow/local/volume）和四种图元类型（triangle/curve/motion_triangle/point）定义了索引值。三角形使用 HIPRT 内建求交，因此其求交函数索引为 None。
- **`Filter_Function_Table_Index`** - 过滤函数表索引。与求交函数表对应，定义了各图元类型在各光线类型下的过滤函数索引。

### 数据访问宏

- **`kernel_data`** - 映射到 `kernel_params.data`。
- **`kernel_data_fetch(name, index)`** - 按索引从内核参数数组中获取数据。
- **`kernel_data_array(name)`** - 获取内核参数数组指针。
- **`kernel_integrator_state`** - 映射到 `kernel_params.integrator_state`。

## 依赖关系

- **内部头文件**:
  - `kernel/types.h` - 内核基础类型定义
  - `kernel/integrator/state.h` - 积分器状态定义
  - `kernel/util/profiler.h` - 性能分析工具
  - `util/color.h` - 颜色工具
  - `util/texture.h` - 纹理工具
  - `kernel/data_arrays.h` - 内核数据数组声明（通过宏展开）

- **被引用**:
  - `kernel/device/hiprt/kernel.cpp` - 主内核编译入口
  - `device/hiprt/device_impl.cpp` - HIPRT 设备实现
  - `device/hiprt/queue.cpp` - HIPRT 命令队列

## 实现细节 / 关键算法

1. **双层栈设计**: HIPRT 遍历使用全局内存栈（`global_stack_buffer`）和共享内存栈（`shared_stack`）的组合。共享内存栈用于高频访问以提升性能，全局栈作为溢出备份。总共享栈大小为 `HIPRT_THREAD_GROUP_SIZE * HIPRT_SHARED_STACK_SIZE = 6144` 个 int。

2. **线程组大小权衡**: 默认的 1024 线程/组会导致本地内存超限。256 线程/组在内存使用和 GPU 波前占用率之间取得平衡。

3. **函数表二维索引**: 求交/过滤函数表使用 `numGeomTypes * ray_type + geom_type` 的线性索引方式，将图元类型和光线类型的二维组合映射到一维函数表。

4. **HIPRT 专有数据数组**: `user_instance_id` 映射实例 ID 到对象 ID；`blas_ptr` 存储底层加速结构指针；`custom_prim_info` 和 `custom_prim_info_offset` 存储自定义图元（曲线、点云、运动三角形）的索引映射信息。

## 关联文件

- `kernel/device/hiprt/common.h` - 使用本文件定义的枚举和结构体
- `kernel/device/hiprt/bvh.h` - 使用本文件的遍历栈类型和数据访问宏
- `kernel/device/hiprt/hiprt_kernels.h` - 使用 `HIPRT_INIT_KERNEL_GLOBAL` 宏
- `kernel/device/hiprt/kernel.cpp` - 包含本文件作为编译基础
- `kernel/data_arrays.h` - 被本文件宏展开包含
