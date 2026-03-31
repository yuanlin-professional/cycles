# device.h / device.cpp - CPU 设备的创建与信息查询入口

## 概述

本文件对外提供 CPU 渲染设备的工厂函数与能力查询接口，是 Cycles 渲染引擎中 CPU 设备子系统的公共头文件。`device.h` 声明了三个自由函数，`device.cpp` 负责实现它们，包括实例化 `CPUDevice`、填充设备信息以及报告 CPU 指令集能力。

## 核心函数

### `device_cpu_create()`

```cpp
unique_ptr<Device> device_cpu_create(const DeviceInfo &info,
                                     Stats &stats,
                                     Profiler &profiler,
                                     bool headless);
```

- **功能**: 创建并返回一个 `CPUDevice` 实例（通过 `make_unique<CPUDevice>` 构造）。这是 CPU 设备的唯一工厂入口，由上层 `device.cpp` 中的设备创建逻辑调用。
- **参数**:
  - `info` — 设备描述信息（类型、ID、线程数等）
  - `stats` — 内存统计对象
  - `profiler` — 性能分析器
  - `headless` — 是否为无界面模式

### `device_cpu_info()`

```cpp
void device_cpu_info(vector<DeviceInfo> &devices);
```

- **功能**: 将 CPU 设备信息插入到设备列表的头部。填充的信息包括：
  - `type = DEVICE_CPU`
  - `description` — 来自 `system_cpu_brand_string()` 的 CPU 品牌字符串
  - `has_osl = true` — 支持 Open Shading Language
  - `has_nanovdb = true` — 支持 NanoVDB 体积数据
  - `has_profiling = true` — 支持性能分析
  - `has_guiding` — 取决于 `guiding_supported()` 的返回值（路径引导支持）
  - `denoisers` — 若 OpenImageDenoise 可用则添加 `DENOISER_OPENIMAGEDENOISE`

### `device_cpu_capabilities()`

```cpp
string device_cpu_capabilities();
```

- **功能**: 返回当前 CPU 支持的指令集能力字符串。若支持 AVX2 则返回 `"AVX2"`，否则返回空字符串。该信息用于选择最优的内核实现。

## 依赖关系

### 内部头文件
- `device/cpu/device.h` — 本文件自身的头文件声明
- `device/cpu/device_impl.h` — `CPUDevice` 类定义（工厂函数需要它来实例化）
- `device/device.h` — `Device` 基类与 `DeviceInfo` 定义
- `util/guiding.h` — 路径引导设备支持检测
- `util/openimagedenoise.h` — OpenImageDenoise 支持检测
- `util/string.h`、`util/unique_ptr.h`、`util/vector.h` — 基础工具类型

### 被引用
- `src/device/device.cpp` — 设备管理总入口，调用 `device_cpu_create`、`device_cpu_info`、`device_cpu_capabilities`
- `src/integrator/path_trace.cpp` — 路径追踪器中引用 CPU 设备创建
- `src/session/session.cpp` — 渲染会话中引用 CPU 设备信息
- `src/session/denoising.cpp` — 降噪模块中引用 CPU 设备信息

## 实现细节

1. **工厂模式**: `device_cpu_create` 采用简单工厂模式，返回 `unique_ptr<Device>`，调用方无需关心 `CPUDevice` 的具体类型。
2. **设备信息前插**: `device_cpu_info` 使用 `devices.insert(devices.begin(), info)` 将 CPU 设备插入到设备列表最前面，确保 CPU 始终作为首选/默认设备出现。
3. **条件编译**: 路径引导和降噪支持通过运行时检测函数 `guiding_supported()` 和 `openimagedenoise_supported()` 判断，而非编译期宏。

## 关联文件

- `src/device/cpu/device_impl.h` / `device_impl.cpp` — `CPUDevice` 类的完整定义与实现
- `src/device/device.h` / `device.cpp` — `Device` 基类及全局设备管理
- `src/device/cpu/kernel.h` — CPU 内核函数集合（`CPUDevice` 构造时引用）
