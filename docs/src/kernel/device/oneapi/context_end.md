# context_end.h - ONEAPIKernelContext 类定义结束与宏重定向

## 概述

本文件关闭 `ONEAPIKernelContext` 类定义，并重定义 `kernel_integrator_state` 宏，使其通过 `kg` 指针间接访问积分器状态。它与 `context_begin.h` 配对使用，共同完成 oneAPI 内核上下文类的定义。

## 核心函数/宏定义

### 类定义关闭

```cpp
}; /* end of ONEAPIKernelContext class definition */
```

关闭在 `context_begin.h` 中打开的 `ONEAPIKernelContext` 类。

### 宏重定向

```cpp
#undef kernel_integrator_state
#define kernel_integrator_state (*(kg->integrator_state))
```

取消 `globals.h` 中原始的 `kernel_integrator_state` 定义（`(*(integrator_state))`），将其重定向为通过 `kg` 指针访问。这是因为在内核入口点的 lambda 函数中，全局数据通过 `kg` 指针（即 `KernelGlobalsGPU*`）间接访问，而非直接作为成员使用。

## 依赖关系

- **内部头文件**: 无额外包含
- **被引用**:
  - `kernel/device/gpu/kernel.h` — 当 `__KERNEL_ONEAPI__` 定义时，在 GPU 通用内核代码之后包含本文件

## 实现细节 / 关键算法

### 宏重定向的必要性

在 `globals.h` 中，数据访问宏基于"直接成员访问"模式设计（如 `kernel_data` 展开为 `(*(__data))`）。但在内核入口 lambda 中，`KernelGlobalsGPU` 对象通过指针 `kg` 被捕获。此时需要将 `kernel_integrator_state` 调整为指针解引用形式 `(*(kg->integrator_state))`。

### 与 Metal 实现的对比

Metal 的 `context_end.h` 重定向为 `context.launch_params_metal.integrator_state`，而 oneAPI 重定向为 `(*(kg->integrator_state))`。这反映了两个后端不同的数据访问模式：Metal 使用组合关系（context 持有 params 引用），oneAPI 使用继承关系（kg 本身就是 KernelGlobalsGPU）。

## 关联文件

- `kernel/device/oneapi/context_begin.h` — 配对文件，打开 ONEAPIKernelContext 类定义
- `kernel/device/oneapi/globals.h` — 原始 `kernel_integrator_state` 宏定义
- `kernel/device/gpu/kernel.h` — GPU 通用内核框架
