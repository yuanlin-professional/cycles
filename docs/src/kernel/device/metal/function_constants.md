# function_constants.h - Metal 函数常量定义

## 概述

本文件利用 Metal 的函数常量（Function Constants）机制，将 `KernelData` 结构体的各成员暴露为编译期常量。函数常量允许 Metal 着色器编译器在管线创建时根据具体常量值进行优化（如死代码消除、分支折叠），从而在不重新编译源码的情况下生成针对特定场景配置优化的内核变体。

## 核心函数/宏定义

### 常量索引枚举

```cpp
enum {
  Kernel_DummyConstant,
  #define KERNEL_STRUCT_MEMBER(parent, type, name) KernelData_##parent##_##name,
  #include "kernel/data_template.h"
  KernelData_kernel_features
};
```

通过包含 `data_template.h` 自动为 `KernelData` 的每个成员生成唯一的枚举值索引，用作 Metal 函数常量的 ID。`Kernel_DummyConstant` 占位索引 0，`KernelData_kernel_features` 是最后一个特殊常量。

### 函数常量声明

```cpp
constant type kernel_data_##parent##_##name [[function_constant(KernelData_##parent##_##name)]];
```

当 `__KERNEL_METAL__` 定义时，为每个 `KernelData` 成员生成一个 `constant` 限定的 Metal 函数常量变量。这些变量在管线状态对象创建时由主机端设置具体值。

特别地，`kernel_data_kernel_features` 作为 `int` 类型函数常量，控制着内核功能特性的启用/禁用。

## 依赖关系

- **内部头文件**:
  - `kernel/data_template.h` — 通过 X 宏模式（`KERNEL_STRUCT_MEMBER`）提供 `KernelData` 结构体成员列表，被包含两次（枚举生成和常量声明）
- **被引用**:
  - `kernel/device/metal/kernel.metal` — 在 `globals.h` 之后、`gpu/kernel.h` 之前包含
  - `device/metal/kernel.mm` — 主机端设置函数常量值时引用索引枚举

## 实现细节 / 关键算法

### X 宏模式

本文件两次包含 `data_template.h`，每次使用不同的 `KERNEL_STRUCT_MEMBER` 宏定义：

1. **第一次**：生成枚举常量 `KernelData_##parent##_##name`，建立名称到索引的映射
2. **第二次**：使用 `[[function_constant(...)]]` 属性声明实际的 Metal 函数常量变量

### 与主机端的协作

主机端（`device/metal/kernel.mm`）在创建 Metal 管线时，遍历同样的枚举索引，调用 `MTLFunctionConstantValues` 的 `setConstantValue:type:atIndex:` 方法为每个函数常量赋值。这样每个场景配置都能生成最优化的着色器代码。

## 关联文件

- `kernel/data_template.h` — KernelData 成员定义模板
- `kernel/device/metal/kernel.metal` — Metal 内核入口文件
- `kernel/device/metal/globals.h` — Metal 全局数据结构
- `device/metal/kernel.mm` — 主机端 Metal 内核管理（设置函数常量值）
