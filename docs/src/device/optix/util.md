# util.h - OptiX 错误检查工具宏

## 概述

本文件提供 OptiX API 调用的错误检查工具宏，用于在整个 OptiX 设备后端中统一处理 OptiX 函数的返回值。此外，它还负责正确引入 OptiX SDK 头文件，并处理动态加载 CUDA（CUEW）时的兼容性问题。

## 类与结构体

本文件不定义类或结构体。

## 核心函数

本文件不包含函数定义，仅定义预处理器宏。

### `optix_device_assert(optix_device, stmt)`
- **类型**: 预处理器宏
- **功能**: 执行 OptiX API 语句 `stmt`，检查返回值是否为 `OPTIX_SUCCESS`。若失败，通过 `optixGetErrorName()` 获取错误名称，调用 `optix_device->set_error()` 设置格式化的错误信息，包含错误名、语句文本、源文件名和行号。
- **用法**: 用于需要显式指定设备对象的场景。
- **示例展开**:
  ```cpp
  optix_device_assert(some_device, optixPipelineCreate(...));
  // 展开为：
  {
    OptixResult result = optixPipelineCreate(...);
    if (result != OPTIX_SUCCESS) {
      const char *name = optixGetErrorName(result);
      some_device->set_error(
          string_printf("%s in %s (%s:%d)", name, "optixPipelineCreate(...)", __FILE__, __LINE__));
    }
  }
  (void)0
  ```

### `optix_assert(stmt)`
- **类型**: 预处理器宏
- **功能**: `optix_device_assert` 的简写形式，将 `this` 作为设备对象。用于 `OptiXDevice` 类成员函数内部。
- **展开**: `optix_device_assert(this, stmt)`

## 依赖关系

- **内部头文件**:
  - `device/cuda/util.h` — CUDA 错误检查工具（`cuda_assert` 等宏）
- **外部依赖**:
  - `<cuew.h>` — CUDA Extension Wrangler（动态加载 CUDA 时使用，通过 `WITH_CUDA_DYNLOAD` 条件编译）
  - `<optix_stubs.h>` — OptiX API 存根头文件，提供 `optixInit()`、`optixGetErrorName()` 等函数声明
- **被引用**:
  - `src/device/optix/device_impl.h` — `OptiXDevice` 类定义中引入本文件
  - `src/integrator/denoiser_optix.h` — OptiX 降噪器中引入本文件

## 实现细节 / 关键算法

1. **CUEW 兼容处理**: 当定义了 `WITH_CUDA_DYNLOAD` 时，通过 `#define OPTIX_DONT_INCLUDE_CUDA` 阻止 OptiX SDK 头文件包含标准 CUDA 头文件，转而使用 CUEW 提供的动态加载版本。这确保了在不直接链接 CUDA 运行库的环境中正常工作。

2. **末尾 `(void)0` 技巧**: 宏定义末尾的 `(void)0` 使得宏在使用时需要以分号结尾（`optix_assert(stmt);`），与普通函数调用语法一致，避免编译器警告和潜在的语法问题。

3. **错误信息格式**: 错误信息格式为 `"ErrorName in statement (file:line)"`，包含完整的调试定位信息，便于在日志中快速定位 OptiX API 调用失败的位置。

## 关联文件

- `src/device/cuda/util.h` — CUDA 层的对应错误检查工具
- `src/device/optix/device_impl.h` / `device_impl.cpp` — 本宏的主要使用者
- `src/device/optix/queue.cpp` — 在 `optixLaunch()` 调用中使用本宏
- `src/integrator/denoiser_optix.h` / `denoiser_optix.cpp` — 降噪器中使用本宏
