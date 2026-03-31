# kernel.cpp - HIPRT 设备内核编译入口

## 概述

本文件是 HIPRT 后端的主编译单元，负责将所有必要的头文件按正确顺序组装，生成可在 AMD GPU 上运行的 HIPRT 加速光线追踪内核。文件内容非常精简，仅包含必要的 `#include` 指令，受 `__HIP_DEVICE_COMPILE__` 宏保护以确保仅在设备编译时生效。

## 核心函数/宏定义

本文件不定义任何函数或宏，仅作为编译入口组织头文件包含顺序：

1. **`kernel/device/hip/compat.h`** - HIP 设备兼容层，定义 HIP 平台相关的设备函数修饰符和类型映射。
2. **`kernel/device/hip/config.h`** - HIP 设备配置，定义线程块大小等编译配置。
3. **`<hiprt/impl/hiprt_device_impl.h>`** - HIPRT 库的设备端实现头文件，提供遍历器和栈等 HIPRT 核心 API。
4. **`kernel/device/hiprt/globals.h`** - HIPRT 全局数据结构和内核参数。
5. **`kernel/device/gpu/image.h`** - GPU 纹理/图像采样函数。
6. **`kernel/device/gpu/kernel.h`** - 通用 GPU 内核定义（其中条件包含 `hiprt_kernels.h`）。

## 依赖关系

- **内部头文件**:
  - `kernel/device/hip/compat.h` - HIP 兼容层
  - `kernel/device/hip/config.h` - HIP 配置
  - `kernel/device/hiprt/globals.h` - HIPRT 全局定义
  - `kernel/device/gpu/image.h` - GPU 图像处理
  - `kernel/device/gpu/kernel.h` - 通用 GPU 内核

- **外部头文件**:
  - `<hiprt/impl/hiprt_device_impl.h>` - HIPRT SDK 设备端实现

- **被引用**: 本文件作为独立编译单元，由构建系统直接编译，不被其他源文件包含。

## 实现细节 / 关键算法

1. **编译保护**: `__HIP_DEVICE_COMPILE__` 宏由 HIP 编译器（hipcc）在设备代码编译阶段自动定义。整个文件内容被此宏包裹，确保只在设备编译通道中生效，主机编译时本文件为空。

2. **头文件顺序关键性**: 包含顺序严格要求：先加载 HIP 兼容层和配置，再加载 HIPRT 设备实现（因其依赖 HIP 基础类型），然后才是 Cycles 内核全局定义和 GPU 通用内核。任何顺序错乱都会导致编译失败。

3. **编译产物**: 此文件编译生成的目标代码包含所有 HIPRT 内核函数（来自 `hiprt_kernels.h`）以及 HIPRT 自定义求交和过滤回调函数（来自 `common.h`）。

## 关联文件

- `kernel/device/hip/compat.h` - HIP 平台兼容定义
- `kernel/device/hip/config.h` - 编译配置
- `kernel/device/hiprt/globals.h` - 全局数据结构
- `kernel/device/hiprt/common.h` - 自定义求交/过滤函数（间接包含）
- `kernel/device/hiprt/bvh.h` - BVH 求交实现（间接包含）
- `kernel/device/hiprt/hiprt_kernels.h` - HIPRT 内核入口点（间接包含）
- `kernel/device/gpu/kernel.h` - 通用 GPU 内核
