# profiler.h - 内核性能分析宏定义

## 概述
本文件为 Cycles 渲染内核提供性能分析（Profiling）的宏接口。它在 CPU 端包装了 `util/profiling.h` 中的 `ProfilingHelper` 和 `ProfilingWithShaderHelper` 类，在 GPU 端则将所有宏定义为空操作（no-op），因为 GPU 端使用独立的性能分析工具。这种设计使得内核代码可以统一使用相同的宏接口而无需区分运行平台。

## 类与结构体
本文件未定义类或结构体。使用的类来自 `util/profiling.h`：
- **`ProfilingHelper`** — CPU 端通用性能分析辅助类，追踪事件时间
- **`ProfilingWithShaderHelper`** — CPU 端着色器专用性能分析辅助类，额外追踪着色器和物体信息

## 枚举与常量
本文件未定义枚举或常量。

## 核心函数
本文件不包含函数，仅定义以下预处理器宏：

### PROFILING_INIT(kg, event)
- **功能**: 初始化性能分析。在 CPU 端创建一个 `ProfilingHelper` 实例并设置初始事件；在 GPU 端展开为空。
- **参数**: `kg` — KernelGlobals 指针，`event` — 初始分析事件类型

### PROFILING_EVENT(event)
- **功能**: 记录一个性能分析事件。在 CPU 端调用 `profiling_helper.set_event(event)`；在 GPU 端展开为空。
- **参数**: `event` — 分析事件类型

### PROFILING_INIT_FOR_SHADER(kg, event)
- **功能**: 初始化着色器级别的性能分析。在 CPU 端创建一个 `ProfilingWithShaderHelper` 实例；在 GPU 端展开为空。
- **参数**: `kg` — KernelGlobals 指针，`event` — 初始分析事件类型

### PROFILING_SHADER(object, shader)
- **功能**: 记录当前正在处理的着色器。在 CPU 端调用 `profiling_helper.set_shader(object, shader & SHADER_MASK)`；在 GPU 端展开为空。
- **参数**: `object` — 物体标识，`shader` — 着色器索引（经过 `SHADER_MASK` 掩码处理）

## 依赖关系
- **内部头文件**:
  - `util/profiling.h` — 仅 CPU 端（`__KERNEL_GPU__` 未定义时）引入，提供 `ProfilingHelper` 和 `ProfilingWithShaderHelper` 类的实现
- **被引用**: 本文件被所有设备后端的全局定义文件引用：
  - `kernel/device/cpu/globals.h` — CPU 后端
  - `kernel/device/cuda/globals.h` — CUDA 后端
  - `kernel/device/hip/globals.h` — HIP 后端
  - `kernel/device/hiprt/globals.h` — HIPRT 后端
  - `kernel/device/metal/globals.h` — Metal 后端
  - `kernel/device/oneapi/globals.h` — oneAPI 后端
  - `kernel/device/optix/globals.h` — OptiX 后端

## 实现细节 / 关键算法
- 通过预处理条件 `#ifndef __KERNEL_GPU__` 实现平台差异化：CPU 端宏展开为实际的性能分析代码，GPU 端宏展开为空语句。
- 所有宏均使用局部变量名 `profiling_helper`，因此 `PROFILING_INIT` / `PROFILING_INIT_FOR_SHADER` 创建的实例可被同一作用域内的 `PROFILING_EVENT` / `PROFILING_SHADER` 宏直接引用。
- `PROFILING_SHADER` 宏中对 shader 参数应用了 `SHADER_MASK` 位掩码，以提取纯着色器索引（去除附加标志位）。

## 关联文件
- `src/util/profiling.h` — CPU 端性能分析基础设施的实际实现
- `src/kernel/device/cpu/globals.h` — CPU 设备全局定义，引入本文件
- `src/kernel/integrator/` — 积分器代码中广泛使用这些宏进行路径追踪各阶段的性能分析
