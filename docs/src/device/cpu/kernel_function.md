# kernel_function.h - CPU 内核函数的微架构分发包装器

## 概述

本文件定义了 `CPUKernelFunction` 模板类，它是 Cycles 渲染引擎中 CPU 内核函数的运行时分发机制。该类封装了同一内核在不同 CPU 微架构（默认实现与 AVX2 优化实现）下的函数指针，在构造时根据当前 CPU 的指令集支持情况自动选择最优版本，并通过 `operator()` 提供透明的函数调用接口。

## 类与结构体

### CPUKernelFunction\<FunctionType\>

- **继承**: 无（独立模板类）
- **功能**: 包装一对不同微架构的内核函数指针，运行时自动选择最优实现，对外暴露统一的函数调用接口。

#### 模板参数

- `FunctionType` — 内核函数的函数指针类型（如 `void (*)(const ThreadKernelGlobalsCPU*, IntegratorStateCPU*)`）

#### 关键成员

| 成员 | 类型 | 说明 |
|------|------|------|
| `kernel_info_` | `KernelInfo` | 存储当前选中的最优内核信息（函数指针 + 微架构名称） |

#### 关键方法

| 方法 | 说明 |
|------|------|
| `CPUKernelFunction(kernel_default, kernel_avx2)` | 构造函数。接收默认实现和 AVX2 实现两个函数指针，调用 `get_best_kernel_info` 选择最优版本 |
| `operator()(args...)` | 函数调用运算符。使用完美转发将参数传递给选中的内核函数并返回结果。调用前通过 `assert` 确保内核指针有效 |
| `get_uarch_name()` | 返回当前选中的微架构名称字符串（`"AVX2"` 或 `"default"`） |

#### 内部类 KernelInfo

| 成员 | 类型 | 说明 |
|------|------|------|
| `uarch_name` | `const char*` | 微架构名称（用于日志输出） |
| `kernel` | `FunctionType` | 实际的内核函数指针 |

#### 保护方法

| 方法 | 说明 |
|------|------|
| `get_best_kernel_info(kernel_default, kernel_avx2)` | 核心选择逻辑。检查 AVX2 编译支持和运行时 CPU 能力，返回最优的 `KernelInfo` |

## 核心函数

### `get_best_kernel_info()`

```cpp
KernelInfo get_best_kernel_info(FunctionType kernel_default, FunctionType kernel_avx2)
```

微架构选择的核心算法：

1. 若编译时启用了 `WITH_CYCLES_OPTIMIZED_KERNEL_AVX2` 宏：
   - 同时检查 `DebugFlags().cpu.has_avx2()`（调试标志未禁用 AVX2）和 `system_cpu_support_avx2()`（CPU 硬件支持 AVX2）
   - 两个条件都满足时，返回 AVX2 实现
2. 否则回退到默认实现

## 依赖关系

### 内部头文件
- `util/debug.h` — `DebugFlags` 类，用于检查调试标志中是否禁用了特定指令集
- `util/system.h` — `system_cpu_support_avx2()` 函数，运行时检测 CPU 是否支持 AVX2 指令集

### 被引用
- `src/device/cpu/kernel.h` — `CPUKernels` 类中所有内核成员的类型均为 `CPUKernelFunction` 的特化实例

## 实现细节 / 关键算法

### 微架构分发机制

`CPUKernelFunction` 采用"构造时决策，调用时零开销"的策略：

1. **构造时**: 通过 `get_best_kernel_info` 一次性确定最优内核实现，将函数指针存储到 `kernel_info_` 中。
2. **调用时**: `operator()` 直接调用已缓存的函数指针，没有任何分支判断开销。

这种设计使得内核分发的成本仅发生在程序初始化阶段（`CPUKernels` 构造时），后续每次调用都是零额外开销的直接函数指针调用。

### 双重检查机制

AVX2 的启用需要同时满足两个条件：
- **编译期**: `WITH_CYCLES_OPTIMIZED_KERNEL_AVX2` 宏必须启用（表示 AVX2 优化内核已被编译到二进制中）
- **运行时**: `DebugFlags().cpu.has_avx2()` 返回 `true`（用户未通过调试标志禁用 AVX2）且 `system_cpu_support_avx2()` 返回 `true`（CPU 硬件确实支持 AVX2）

这种设计允许开发者在调试时强制禁用 AVX2 优化，以便排查指令集相关的问题。

### 可变参数转发

`operator()` 使用 C++ 可变参数模板和完美转发：

```cpp
template<typename... Args> auto operator()(Args... args) const
{
    return kernel_info_.kernel(args...);
}
```

这使得同一个 `CPUKernelFunction` 实例可以适配任意函数签名，只要签名与 `FunctionType` 一致即可。

## 关联文件

- `src/device/cpu/kernel.h` / `kernel.cpp` — `CPUKernels` 类，使用本模板定义和注册所有 CPU 内核
- `src/util/debug.h` — 调试标志系统，控制指令集启用/禁用
- `src/util/system.h` — 系统能力检测，运行时查询 CPU 指令集支持
