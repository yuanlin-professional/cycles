# context_intersect_end.h - oneAPI Embree GPU 求交上下文结束

## 概述

本文件恢复 `ccl_gpu_kernel_signature` 宏为标准定义，撤销 `context_intersect_begin.h` 中为 Embree GPU 求交内核所做的修改。它与 `context_intersect_begin.h` 配对使用，确保非求交内核不会携带多余的 `sycl::kernel_handler` 参数。

## 核心函数/宏定义

### 宏恢复

```cpp
#undef ccl_gpu_kernel_signature
#define ccl_gpu_kernel_signature __ccl_gpu_kernel_signature
```

将 `ccl_gpu_kernel_signature` 恢复为 `compat.h` 中定义的标准版本 `__ccl_gpu_kernel_signature`，该版本不包含 `sycl::kernel_handler` 参数。

### 条件编译

仅在以下条件同时满足时生效：
- 未定义 `WITH_ONEAPI_SYCL_HOST_TASK`
- 定义了 `WITH_EMBREE_GPU`

当条件不满足时，本文件无任何效果。

## 依赖关系

- **内部头文件**: 无额外包含
- **被引用**:
  - `kernel/device/gpu/kernel.h` — 在包含 Embree 求交相关内核代码之后包含本文件

## 实现细节 / 关键算法

### 与 context_intersect_begin.h 的配对

在 `kernel/device/gpu/kernel.h` 中的使用模式：

```
// 求交内核前
#include "kernel/device/oneapi/context_intersect_begin.h"
// ... 包含 integrator_intersect_closest 等求交内核 ...
#include "kernel/device/oneapi/context_intersect_end.h"
// 非求交内核继续使用标准签名
```

这种机制确保只有确实需要 Embree 硬件光线追踪功能的内核才会接收 `kernel_handler` 参数，避免对其他内核造成不必要的性能影响。

## 关联文件

- `kernel/device/oneapi/context_intersect_begin.h` — 配对文件，设置求交内核签名
- `kernel/device/oneapi/compat.h` — 标准 `__ccl_gpu_kernel_signature` 定义
- `kernel/device/gpu/kernel.h` — GPU 通用内核框架
