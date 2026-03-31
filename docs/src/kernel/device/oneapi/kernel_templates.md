# kernel_templates.h - oneAPI 内核调用类型安全模板

## 概述

本文件通过预处理器宏魔法生成一系列 C++ 模板函数 `oneapi_call()`，实现从 `void**` 参数数组到具体类型内核函数的类型安全调用。由于 Cycles 的设备内核参数统一以 `void**` 形式传递，本文件的模板自动将每个参数从 `void*` 转型为内核函数签名所期望的类型，是 oneAPI 内核调度机制的核心组件。

## 核心函数/宏定义

### 生成的模板函数

```cpp
template<typename T0, typename T1, ...>
void oneapi_call(
    KernelGlobalsGPU *kg,
    sycl::handler &cgh,
    size_t global_size,
    size_t local_size,
    void **args,
    void (*func)(KernelGlobalsGPU*, size_t, size_t, sycl::handler&, T0, T1, ...))
{
    func(kg, global_size, local_size, cgh,
         *(T0*)(args[0]), *(T1*)(args[1]), ...);
}
```

通过 `oneapi_template(0)` 到 `oneapi_template(0, 1, ..., 20)` 共生成 21 个模板重载，支持 1 到 21 个内核参数。

### 核心宏定义

| 宏 | 说明 |
|----|------|
| `ONEAPI_TYP(x)` | 生成模板参数 `typename Tx` |
| `ONEAPI_CAST(x)` | 生成参数转型 `*(Tx*)(args[x])` |
| `ONEAPI_T(x)` | 生成类型引用 `Tx` |
| `ONEAPI_CALL_FOR(x, ...)` | 可变参数宏分发器，根据参数数量调用对应的展开宏 |
| `ONEAPI_GET_NTH_ARG(...)` | 参数计数辅助宏，选择正确的展开层级 |
| `ONEAPI_0` - `ONEAPI_21` | 递归展开宏，将操作应用到每个参数索引 |
| `oneapi_template(...)` | 最终模板生成宏 |

### 宏展开示例

`oneapi_template(0, 1, 2)` 展开为：

```cpp
template<typename T0, typename T1, typename T2>
void oneapi_call(
    KernelGlobalsGPU *kg,
    sycl::handler &cgh,
    size_t global_size,
    size_t local_size,
    void **args,
    void (*func)(KernelGlobalsGPU*, size_t, size_t, sycl::handler&, T0, T1, T2))
{
    func(kg, global_size, local_size, cgh,
         *(T0*)(args[0]), *(T1*)(args[1]), *(T2*)(args[2]));
}
```

## 依赖关系

- **内部头文件**: 无（纯宏定义）
- **被引用**:
  - `kernel/device/oneapi/kernel.cpp` — 在 `oneapi_enqueue_kernel()` 中通过 `oneapi_call()` 调用各种内核

## 实现细节 / 关键算法

### 可变参数宏计数技术

使用经典的"计数反向选择"技巧：

```cpp
#define ONEAPI_GET_NTH_ARG(_1,...,_22, N, ...) N
#define ONEAPI_CALL_FOR(x, ...) ONEAPI_GET_NTH_ARG("ignored", ##__VA_ARGS__,
    ONEAPI_21, ONEAPI_20, ..., ONEAPI_1, ONEAPI_0)(x, ##__VA_ARGS__)
```

当传入 N 个参数时，`ONEAPI_GET_NTH_ARG` 将第 N+1 个位置（从末尾数）的 `ONEAPI_N` 宏选出，然后用该宏递归展开所有参数。

### 类型安全保障

虽然参数以 `void**` 传递，但 C++ 模板类型推导确保 `oneapi_call` 的最后一个参数（函数指针）的类型签名与实际内核函数匹配。如果参数数量不匹配，编译器会报错。转型本身（`*(Tx*)(args[x])`）仍然是不安全的强制转换，但至少类型与数量得到了编译期保证。

### 参数数量限制

支持最多 21 个内核参数（`oneapi_template(0, 1, ..., 20)`）。这对当前 Cycles 所有内核来说是足够的。如需扩展，只需增加 `ONEAPI_N` 宏定义和 `oneapi_template` 调用。

### 与 Metal 实现的对比

Metal 后端使用 `PARAMS_MAKER` / `FN0-FN20` 宏系列生成结构体成员，通过结构体传递参数。oneAPI 使用函数指针模板进行类型安全转型。两者解决的是同一问题（从通用参数数组到类型化调用），但采用了不同的技术路径。

## 关联文件

- `kernel/device/oneapi/kernel.cpp` — 使用 `oneapi_call()` 的内核调度代码
- `kernel/device/oneapi/compat.h` — 定义内核入口函数签名（与函数指针类型匹配）
- `kernel/device/gpu/kernel.h` — GPU 通用内核定义（生成被调用的内核函数）
