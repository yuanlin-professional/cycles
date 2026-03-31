# device.h / device.cpp - oneAPI 设备注册与工厂入口

## 概述

本文件是 Cycles 渲染器 oneAPI 设备后端的对外入口层，提供设备初始化、创建、枚举和能力查询四个顶层函数。它作为 oneAPI 子系统与 Cycles 通用设备框架（`device/device.h`）之间的桥梁，将具体实现委托给 `OneapiDevice`（定义于 `device_impl.h`）。当编译时未启用 `WITH_ONEAPI` 宏时，所有函数均退化为空操作或返回失败。

## 类与结构体

本文件不定义类，仅声明和实现四个命名空间级自由函数以及一个文件作用域的静态回调函数。

## 核心函数

### `device_oneapi_init()`
- **签名**: `bool device_oneapi_init()`
- **功能**: 初始化 oneAPI 运行时环境。通过设置一系列环境变量来配置 SYCL/Level-Zero 行为：
  - `SYCL_CACHE_PERSISTENT=1` / `SYCL_CACHE_THRESHOLD=0` — 启用 JIT 编译缓存，提升后续启动速度。
  - `ONEAPI_DEVICE_SELECTOR` — 默认仅启用 Level-Zero 后端；若设置了 `CYCLES_ONEAPI_ALL_DEVICES`，则同时启用 CUDA/HIP 后端，但始终排除 OpenCL。
  - `ZES_ENABLE_SYSMAN=1` — 启用 SysMan 接口以支持显存空闲容量查询。
  - `SYCL_PI_LEVEL_ZERO_USE_COPY_ENGINE=0` — 禁用拷贝引擎以提高稳定性。
  - 所有环境变量遵循"不覆盖用户预设"原则。
- **返回**: 成功返回 `true`；未编译 oneAPI 支持时返回 `false`。
- **平台差异**: Windows 使用 `_putenv_s()`，Linux 使用 `setenv()`。

### `device_oneapi_create()`
- **签名**: `unique_ptr<Device> device_oneapi_create(const DeviceInfo &info, Stats &stats, Profiler &profiler, bool headless)`
- **功能**: 工厂函数，构造并返回一个 `OneapiDevice` 实例。这是 Cycles 设备框架创建 oneAPI 设备的唯一入口。
- **返回**: 指向 `OneapiDevice` 的 `unique_ptr`；未编译 oneAPI 支持时触发 `LOG_FATAL` 并返回 `nullptr`。

### `device_oneapi_info()`
- **签名**: `void device_oneapi_info(vector<DeviceInfo> &devices)`
- **功能**: 枚举系统中所有可用的 oneAPI 设备，通过回调函数 `device_iterator_cb` 将设备信息填充到 `devices` 向量中。
- **回调 `device_iterator_cb`**: 静态函数，接收设备 ID、名称、编号、硬件光线追踪支持、OIDN 降噪支持及执行优化标志，构造 `DeviceInfo` 结构体并追加到设备列表。其中：
  - `info.type` 设置为 `DEVICE_ONEAPI`
  - `info.has_nanovdb` 始终为 `true`
  - `info.has_peer_memory` 始终为 `false`（oneAPI 当前不支持多设备对等访问）
  - `info.has_gpu_queue` 始终为 `true`
  - 硬件光线追踪支持取决于 `WITH_EMBREE_GPU` 编译选项和运行时检测
  - OIDN 降噪支持取决于 OIDN 版本及设备兼容性

### `device_oneapi_capabilities()`
- **签名**: `string device_oneapi_capabilities()`
- **功能**: 返回所有可用 oneAPI 设备的详细能力信息字符串，委托给 `OneapiDevice::device_capabilities()` 静态方法。

## 依赖关系

### 内部头文件
- `device/oneapi/device.h` — 本文件自身的头文件声明
- `device/device.h` — Cycles 通用设备框架
- `device/oneapi/device_impl.h` — `OneapiDevice` 类定义（条件编译）
- `integrator/denoiser_oidn_gpu.h` — GPU OIDN 降噪器（条件编译）
- `util/log.h`, `util/string.h`, `util/unique_ptr.h`, `util/vector.h`

### 被引用
- `src/device/device.cpp` — Cycles 主设备管理模块通过此文件注册 oneAPI 后端
- `src/device/oneapi/device.cpp` — 自身实现文件

## 实现细节 / 关键算法

1. **环境变量防覆盖机制**: 所有 `setenv` / `_putenv_s` 调用都先检查 `getenv()` 返回值是否为 `nullptr`，仅在用户未预设的情况下写入默认值。这使高级用户可以通过预设环境变量来自定义 SYCL 运行时行为。

2. **条件编译隔离**: 整个实现被 `#ifdef WITH_ONEAPI` 包裹。未启用时，`device_oneapi_create()` 使用 `(void)param` 技巧消除未使用参数警告，并通过 `LOG_FATAL` 报告致命错误。

3. **OIDN 版本分支**: 根据 `OIDN_VERSION >= 20300` 选择不同的设备支持检测路径——新版本使用运行时回调参数 `oidn_support`，旧版本回退到 `OIDNDenoiserGPU::is_device_supported()` 静态方法。

## 关联文件
- `src/device/oneapi/device_impl.h` / `device_impl.cpp` — `OneapiDevice` 类的完整定义与实现
- `src/device/device.h` / `device.cpp` — Cycles 通用设备抽象层
- `src/device/oneapi/queue.h` / `queue.cpp` — oneAPI 设备队列实现
