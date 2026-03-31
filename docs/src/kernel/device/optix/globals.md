# globals.h - OptiX 设备全局数据结构与内核参数定义

## 概述

本文件定义了 OptiX 后端的全局常量数据结构，包括 GPU 内核全局状态 `KernelGlobalsGPU`、内核启动参数结构 `KernelParamsOptiX`，以及数据访问抽象宏。与 HIPRT 后端不同，OptiX 的 `KernelGlobals` 是一个未使用的空指针（由编译器优化掉），所有内核数据通过 `__constant__` 内存中的 `kernel_params` 全局变量访问。

## 核心函数/宏定义

### 结构体

- **`KernelGlobalsGPU`** - GPU 内核全局状态，仅包含一个未使用的 `int unused[1]` 成员。OptiX 不需要像 HIPRT 那样在 `KernelGlobals` 中存储遍历栈，因为 OptiX 内部管理自己的 BVH 遍历栈。此结构体存在仅是为了满足内核函数签名中 `KernelGlobals kg` 参数的类型要求。

- **`KernelParamsOptiX`** - OptiX 内核启动参数结构，包含：
  - `const int *path_index_array` - 路径索引数组指针
  - `float *render_buffer` - 渲染输出缓冲区
  - `int offset` - 着色器评估偏移量
  - `int num_tiles` - 图块数量（相机初始化内核使用）
  - `int max_tile_work_size` - 单个图块最大工作量
  - `KernelData data` - 全局场景数据
  - 内核数据数组（通过 `kernel/data_arrays.h` 宏展开）
  - `IntegratorStateGPU integrator_state` - 积分器 GPU 状态
  - `void *osl_colorsystem` - 开放着色语言（OSL）颜色系统指针

### 类型别名

- **`KernelGlobals`** - `const ccl_global KernelGlobalsGPU *ccl_restrict`，实际使用中始终传递 `nullptr`。

### 全局变量

- **`kernel_params`** - `__constant__` 内存中的 `KernelParamsOptiX` 实例。在非 RDC（Relocatable Device Code）模式下声明为 `static`，在 RDC 模式下为 `extern "C"` 链接。

### 数据访问宏

- **`kernel_data`** - 映射到 `kernel_params.data`。
- **`kernel_data_array(name)`** - 获取内核参数数组指针 `kernel_params.name`。
- **`kernel_data_fetch(name, index)`** - 按索引从内核参数数组中获取数据 `kernel_params.name[(index)]`。
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
  - `kernel/device/optix/kernel.cu` - OptiX 主内核
  - `kernel/device/optix/kernel_osl_camera.cu` - OSL 相机内核
  - `kernel/osl/services_optix.cu` - OSL 服务
  - `device/optix/device_impl.cpp` - OptiX 设备实现
  - `device/optix/queue.cpp` - OptiX 命令队列

## 实现细节 / 关键算法

1. **空 KernelGlobals 设计**: OptiX 的 `KernelGlobals` 始终为 `nullptr`。这是一个有意为之的架构决策 -- OptiX 内核通过 `__constant__` 内存中的 `kernel_params` 直接访问所有数据，`KernelGlobals` 参数仅为了保持与其他后端（CPU、CUDA、HIP）统一的函数签名。编译器会优化掉对这个空指针的传递。

2. **__constant__ 内存**: `kernel_params` 存储在 GPU 常量内存中，具有缓存优化特性。OptiX 运行时在内核启动前通过 `optixLaunch` 将参数数据复制到常量内存。

3. **RDC 模式兼容**: 当使用 `__CUDACC_RDC__`（可重定位设备代码）编译时，`kernel_params` 不使用 `static` 链接，以支持跨编译单元引用（OSL 等场景需要此功能）。

4. **OSL 颜色系统**: `osl_colorsystem` 指针用于开放着色语言（OSL）的颜色空间转换，仅在 OSL 着色器编译时使用。

## 关联文件

- `kernel/device/optix/compat.h` - 兼容层，通常与本文件配对包含
- `kernel/device/optix/kernel.cu` - 主内核编译入口
- `kernel/device/optix/bvh.h` - BVH 求交实现（通过数据访问宏访问内核数据）
- `kernel/data_arrays.h` - 被本文件宏展开包含
- `device/optix/device_impl.cpp` - 主机端设备实现，负责设置 `kernel_params` 的值
