# util.h / util.cpp - HIP 工具函数与上下文管理

## 概述

本文件提供 HIP 后端的基础工具设施，包括 RAII 上下文管理类 `HIPContextScope`、错误检查宏、驱动版本兼容性检测以及设备架构查询辅助函数。这些工具被 HIP 后端的所有其他文件广泛使用，是整个 HIP 子系统的基础层。

## 类与结构体

### `HIPContextScope`
- **功能**: RAII 风格的 HIP 上下文作用域管理器，在构造时将设备上下文推入当前线程的上下文栈，在析构时弹出
- **关键成员**:
  - `HIPDevice *device` — 关联的 HIP 设备指针
- **关键方法**:
  - `HIPContextScope(HIPDevice *device)` — 构造函数，调用 `hipCtxPushCurrent(device->hipContext)`
  - `~HIPContextScope()` — 析构函数，调用 `hipCtxPopCurrent(nullptr)`

## 核心函数

### 宏定义

#### `hip_device_assert(hip_device, stmt)`
- **功能**: 执行 HIP API 调用并检查返回值，失败时通过设备对象报告错误
- **错误信息格式**: `"{错误名} in {语句} ({文件}:{行号})"`
- **使用方式**: 需要传入设备指针和 HIP API 语句

#### `hip_assert(stmt)`
- **功能**: `hip_device_assert` 的简写形式，将 `this` 作为设备指针
- **使用场景**: 在 `HIPDevice` 类的成员函数中使用

### 静态内联函数（头文件中定义）

#### `hipDeviceArch(int hipDevId)`
- **返回值**: `std::string` — GPU 架构名称（如 `"gfx1030"`、`"gfx1100"`）
- **功能**: 通过 `hipGetDeviceProperties()` 获取 `gcnArchName` 属性，使用 `strtok()` 截取冒号前的架构名称部分

#### `hipSupportsDevice(int hipDevId)`
- **返回值**: `bool`
- **功能**: 检查设备的计算能力是否达到要求（`major >= 10`），即要求 AMD RDNA 架构及以上
- **说明**: 计算能力 10.x 对应 RDNA 架构，低于此版本的 GCN 架构不被支持

#### `hipIsRDNA2OrNewer(int hipDevId)`
- **返回值**: `bool`
- **功能**: 检查设备是否为 RDNA2 或更新架构（`major > 10` 或 `major == 10 && minor >= 3`）
- **用途**: 用于判断是否可以启用 HIPRT 硬件光线追踪（RDNA1 因 HIPRT 2.5 / HIP SDK 6.3 的 bug 被排除）

#### `hipSupportsDeviceOIDN(int hipDevId)`
- **返回值**: `bool`
- **功能**: 检查设备架构是否在 OIDN GPU 降噪支持列表中
- **支持的架构**: `gfx1030`（RDNA2）、`gfx1100`/`gfx1101`/`gfx1102`（RDNA3）、`gfx1200`/`gfx1201`（RDNA4）

### cpp 文件中的函数

#### `HIPContextScope::HIPContextScope()` / `~HIPContextScope()`
- 构造时推入上下文，析构时弹出，所有操作都通过 `hip_device_assert` 检查错误

#### `hipewErrorString()` （非动态加载模式）
- **条件编译**: `#ifndef WITH_HIP_DYNLOAD`
- **功能**: 当 HIP 为静态链接时提供错误码到字符串的转换（仅返回错误数字，避免代码重复）
- **注意**: 非线程安全（使用 `static string`）

#### `hipewCompilerPath()` （非动态加载模式）
- **返回值**: 编译时定义的 `CYCLES_HIP_HIPCC_EXECUTABLE` 路径

#### `hipewCompilerVersion()` （非动态加载模式）
- **返回值**: 从 `HIP_VERSION` 宏计算的编译器版本号

#### `hipSupportsDriver()`
- **返回值**: `bool`
- **功能**: 检查 HIP 驱动版本是否满足最低要求：
  - Windows: 要求 `>= 60241512`（对应 AMD Adrenalin 24.9.1 驱动）
  - Linux: 要求 `>= 60000000`（HIP Runtime 6.0 及以上，因 HIP Runtime 5.x 会导致 Blender 崩溃）

## 依赖关系
- **内部头文件**:
  - `device/hip/device_impl.h` — `HIPDevice` 类（`HIPContextScope` 访问其 `hipContext` 成员）
  - `hipew.h` — HIP 动态加载包装（条件编译 `WITH_HIP_DYNLOAD`）
  - `<cstring>`, `<string>` — 标准库
- **被引用**: `src/device/hip/device_impl.h`、`src/device/hip/queue.h`、`src/device/hip/graphics_interop.cpp`

## 实现细节 / 关键算法

- **RAII 上下文管理**: HIP 采用与 CUDA 类似的上下文栈模型。`HIPContextScope` 通过构造/析构自动管理上下文的推入和弹出，确保在作用域退出时（包括异常路径）正确恢复上下文状态。这是 HIP 后端线程安全的基础。
- **动态加载 vs 静态链接**: 通过 `WITH_HIP_DYNLOAD` 条件编译宏区分两种模式：
  - 动态加载（默认）：使用 HIPEW 库在运行时加载 HIP 动态库，允许在无 HIP 环境中优雅降级
  - 静态链接：直接链接 HIP 库，此时需要本文件提供 `hipewErrorString()`、`hipewCompilerPath()`、`hipewCompilerVersion()` 的替代实现
- **架构版本映射**: AMD GPU 架构版本号与产品代号的对应关系：
  - `major >= 10`: RDNA 系列（Navi 及以上）
  - `major == 10, minor >= 3`: RDNA2 系列（Navi 2x）
  - `gfx1030`: RDNA2（如 RX 6800/6900 系列）
  - `gfx11xx`: RDNA3（如 RX 7000 系列）
  - `gfx12xx`: RDNA4（如 RX 9000 系列）

## 关联文件
- `src/device/hip/device_impl.h` / `device_impl.cpp` — 所有 HIP API 调用都依赖本文件的上下文管理和错误检查
- `src/device/hip/queue.h` / `queue.cpp` — 使用 `HIPContextScope` 和 `hip_device_assert`
- `src/device/hip/graphics_interop.cpp` — 使用 `HIPContextScope` 和 `hip_device_assert`
- `src/device/hip/device.cpp` — 使用 `hipSupportsDevice()`、`hipSupportsDriver()` 等辅助函数
