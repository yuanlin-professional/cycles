# services_optix.cu - OptiX 设备上 OSL 服务的 GPU 入口编译单元

## 概述

`services_optix.cu` 是 Cycles 在 NVIDIA OptiX 光线追踪设备上启用 Open Shading Language (OSL) 支持的 CUDA 编译单元。该文件的核心作用是将通用 GPU OSL 服务头文件（`services_gpu.h`）在 OptiX 兼容环境下进行编译，并提供一个虚拟的 `__direct_callable__` 入口函数，使得 OptiX 管线能够正确链接 OSL 服务模块。

该文件本身代码极少，但通过引入的头文件链条，将完整的 OSL GPU 服务实现（纹理查询、几何属性访问、着色器全局变量等）编译为 OptiX 可调用程序。

## 核心函数/类

### `__direct_callable__dummy_services()`
- **签名**: `extern "C" __device__ void __direct_callable__dummy_services()`
- **功能**: 空的 OptiX Direct Callable 函数，作为 OSL 服务模块的占位入口
- **说明**: OptiX 的可编程管线要求每个模块至少有一个可调用入口点。此虚拟函数确保编译单元被正确链接到 OptiX 管线中，而实际的 OSL 服务功能通过 `services_gpu.h` 中内联展开的函数提供

## 依赖关系

- **内部头文件**:
  - `kernel/device/optix/compat.h` — OptiX 设备兼容性宏与类型定义
  - `kernel/device/optix/globals.h` — OptiX 设备全局变量（内核数据指针等）
  - `kernel/device/gpu/image.h` — GPU 纹理查询功能（使用 CUDA 原生纹理内部函数）
  - `kernel/osl/services_gpu.h` — OSL GPU 服务的核心实现，包含几何属性查询、纹理采样、着色器全局变量访问等全部功能
- **预处理器宏**:
  - `WITH_OSL` — 在文件开头定义，启用 OSL 相关代码路径
- **被引用**: `src/kernel/CMakeLists.txt` 通过 `cycles_optix_kernel_add()` 宏将其编译为 OptiX PTX 模块 `kernel_optix_osl_services`，使用 `--relocatable-device-code=true` 选项支持设备端链接

## 实现细节

1. **编译模式**: 该文件使用 `--relocatable-device-code=true`（RDC）模式编译，允许设备代码在多个 PTX 模块间链接。这是因为 OSL 服务函数需要被其他 OptiX 内核模块（如 `kernel_optix_osl.cu`）引用
2. **头文件引入顺序**: 使用 `// clang-format off/on` 注释保护头文件包含顺序，因为 OptiX 兼容层（`compat.h`）必须在其他内核头文件之前包含，以确保宏定义和类型别名正确生效
3. **服务实现委托**: 所有实际的 OSL 服务逻辑（如 `get_attribute()`、`texture()`、`get_matrix()` 等）均在 `services_gpu.h` 中实现。该文件仅负责提供 OptiX 编译环境和入口点
4. **与 CPU OSL 的对比**: CPU 端的 OSL 服务由 `services.cpp` 实现，继承自 OSL 的 `RendererServices` 基类。GPU 端则绕过 OSL 的 C++ 类体系，直接提供设备函数实现

## 关联文件

- `src/kernel/osl/services_gpu.h` — OSL GPU 服务的完整实现头文件
- `src/kernel/osl/services.h` / `src/kernel/osl/services.cpp` — CPU 端 OSL 服务实现
- `src/kernel/osl/services_shared.h` — CPU/GPU 共享的 OSL 服务数据结构
- `src/kernel/device/optix/compat.h` — OptiX 设备兼容性层
- `src/kernel/device/optix/globals.h` — OptiX 设备全局变量
- `src/kernel/device/optix/kernel_osl.cu` — OptiX OSL 着色器执行内核
