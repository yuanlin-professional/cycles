# device.h / device.cpp - HIP 设备注册与工厂入口

## 概述

本文件是 Cycles 渲染器 HIP 后端的对外入口层，提供 HIP 设备的初始化检测、设备实例创建、设备信息枚举以及硬件能力查询四大工厂函数。上层 `device/device.cpp` 通过这些函数将 HIP 后端集成到 Cycles 的多后端设备体系中，而不需要直接依赖 HIP 实现细节。

## 类与结构体

本文件不定义类，仅声明和实现四个命名空间级自由函数。

## 核心函数

### `device_hip_init()`
- **返回值**: `bool`
- **功能**: 检测当前系统是否具备可用的 HIP 运行时环境。在动态加载模式 (`WITH_HIP_DYNLOAD`) 下，调用 `hipewInit()` 初始化 HIPEW 包装层，并依次检查：
  1. 驱动版本是否足够新（通过 `hipSupportsDriver()`）
  2. 是否存在预编译内核 (`HIPDevice::have_precompiled_kernels()`)
  3. 是否能找到 HIPCC 编译器 (`hipewCompilerPath()`)
- **错误处理**: 对 `HIPEW_ERROR_ATEXIT_FAILED`、`HIPEW_ERROR_OLD_DRIVER` 以及动态库加载失败分别输出不同日志警告。使用 `static` 局部变量保证仅初始化一次。

### `device_hip_create()`
- **签名**: `unique_ptr<Device> device_hip_create(const DeviceInfo &info, Stats &stats, Profiler &profiler, bool headless)`
- **功能**: 设备工厂函数。根据编译选项创建具体设备实例：
  - `WITH_HIPRT` 且 `info.use_hardware_raytracing` 为真时，创建 `HIPRTDevice`（硬件光线追踪加速）
  - 否则创建标准 `HIPDevice`
  - 未编译 HIP 支持时触发 `LOG_FATAL`

### `device_hip_info()`
- **签名**: `void device_hip_info(vector<DeviceInfo> &devices)`
- **功能**: 枚举系统中所有可用的 AMD HIP GPU 设备，填充 `DeviceInfo` 结构体列表。对每个设备执行以下操作：
  - 通过 `hipSupportsDevice()` 过滤不支持的设备（要求 RDNA 及以上架构）
  - 检测 P2P（点对点）内存访问能力
  - 检测 HIPRT 硬件光线追踪支持（要求 RDNA2 及以上）
  - 检测 OpenImageDenoise (OIDN) GPU 降噪支持
  - 读取 PCI 总线地址生成唯一设备 ID
  - 检测内核执行超时属性，区分显示设备和计算设备（显示设备排列在列表末尾）
  - 禁用 MNEE（多次嵌套事件估算），因 AMD GPU 编译器 bug 导致停滞或渲染瑕疵

### `device_hip_capabilities()`
- **签名**: `string device_hip_capabilities()`
- **功能**: 以可读文本格式返回所有 HIP 设备的硬件属性报告，包括最大线程数、共享内存大小、多处理器数量、缓存大小等约 30 项 `hipDeviceAttribute` 参数。

### `device_hip_safe_init()` (静态内部函数)
- **功能**: 在 Windows 平台使用 SEH (`__try/__except`) 包裹 `hipInit(0)` 调用，防止损坏的 HIP 驱动导致整个进程崩溃。Linux 平台直接调用 `hipInit(0)`。

## 依赖关系
- **内部头文件**:
  - `device/hip/device.h` — 本模块声明
  - `device/device.h` — 通用设备基类
  - `device/hip/device_impl.h` — `HIPDevice` 实现（条件编译）
  - `device/hiprt/device_impl.h` — `HIPRTDevice` 实现（条件编译 `WITH_HIPRT`）
  - `integrator/denoiser_oidn_gpu.h` — OIDN GPU 降噪器
  - `util/log.h`, `util/string.h`, `util/unique_ptr.h`, `util/vector.h`
- **被引用**: `src/device/device.cpp`（Cycles 总设备管理器通过此文件注册 HIP 后端）

## 实现细节 / 关键算法

- **条件编译层次**: 文件使用三层条件编译宏：`WITH_HIP`（HIP 总开关）、`WITH_HIP_DYNLOAD`（动态加载 vs 静态链接）、`WITH_HIPRT`（HIPRT 硬件光追扩展）。
- **单例初始化**: `device_hip_init()` 使用 `static bool initialized` 确保 HIPEW 初始化过程只执行一次。
- **设备排序策略**: `device_hip_info()` 将带内核执行超时（连接显示器）的设备放在列表末尾，优先返回纯计算设备。

## 关联文件
- `src/device/device.cpp` — 调用本文件函数注册 HIP 后端
- `src/device/hip/device_impl.h` / `device_impl.cpp` — `HIPDevice` 类完整实现
- `src/device/hiprt/device_impl.h` — HIPRT 硬件光追设备实现
- `src/device/hip/util.h` — `hipSupportsDevice()`、`hipSupportsDriver()` 等辅助函数
