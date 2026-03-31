# context_end.h - MetalKernelContext 类定义结束与宏重定向

## 概述

本文件关闭 `MetalKernelContext` 类定义，并重定义 `kernel_integrator_state` 宏，使其通过上下文对象访问积分器状态。它与 `context_begin.h` 配对使用，共同完成 Metal 内核上下文类的定义。

## 核心函数/宏定义

### 类定义关闭

```cpp
}; /* end of MetalKernelContext class definition */
```

关闭在 `context_begin.h` 中打开的 `MetalKernelContext` 类。

### 宏重定向

```cpp
#undef kernel_integrator_state
#define kernel_integrator_state context.launch_params_metal.integrator_state
```

取消 `globals.h` 中原始的 `kernel_integrator_state` 定义，将其重定向为通过 `context` 对象（即 `MetalKernelContext` 实例）访问。这是因为在内核入口点的 `run` 方法中，全局数据通过上下文对象间接访问。

## 依赖关系

- **内部头文件**: 无额外包含
- **被引用**:
  - `kernel/device/gpu/kernel.h` — 当 `__KERNEL_METAL__` 定义时，在 GPU 通用内核代码之后包含本文件

## 实现细节 / 关键算法

### 宏重定向的必要性

在 `globals.h` 中，`kernel_integrator_state` 被定义为 `launch_params_metal.integrator_state`，这适用于直接访问全局参数的场景。但在 `MetalKernelContext::run()` 方法（由 `ccl_gpu_kernel_signature` 宏生成）中，参数通过 `context` 局部变量访问，因此需要将宏重定向为 `context.launch_params_metal.integrator_state`。

## 关联文件

- `kernel/device/metal/context_begin.h` — 配对文件，打开 MetalKernelContext 类定义
- `kernel/device/metal/globals.h` — 原始 `kernel_integrator_state` 宏定义
- `kernel/device/gpu/kernel.h` — GPU 通用内核框架
