# denoise.h / denoise.cpp - 降噪器参数定义与类型管理

## 概述

本文件定义了 Cycles 渲染器中降噪器(Denoiser)的参数体系。它声明了降噪器类型枚举（OptiX 和 OpenImageDenoise）、预过滤模式、质量等级，以及用于序列化/反序列化的 `DenoiseParams` 节点类。该模块是降噪器子系统与场景集成器之间的配置桥梁，不包含实际的降噪算法实现。

## 类与结构体

### DenoiseParams
- **继承**: `Node`（来自 `graph/node.h`）
- **功能**: 封装降噪器的全部配置参数，利用 Node API 实现序列化和反序列化。虽然不是真正的场景节点，但借用了节点系统以简化参数管理。
- **关键成员**:
  - `use` — 是否启用降噪器（`bool`）
  - `type` — 降噪器类型，默认 `DENOISER_OPENIMAGEDENOISE`（`DenoiserType`）
  - `start_sample` — 视口中开始降噪的采样数（`int`）
  - `use_pass_albedo` — 是否使用反照率辅助通道（`bool`）
  - `use_pass_normal` — 是否使用法线辅助通道（`bool`）
  - `temporally_stable` — 是否启用时间稳定模式，利用运动向量和前一帧图像（`bool`）
  - `use_gpu` — 是否允许 OpenImageDenoise 使用 GPU 设备（`bool`）
  - `prefilter` — 预过滤模式，默认 `DENOISER_PREFILTER_FAST`（`DenoiserPrefilter`）
  - `quality` — 降噪质量等级，默认 `DENOISER_QUALITY_HIGH`（`DenoiserQuality`）
- **关键方法**:
  - `get_type_enum()` — 返回降噪器类型的 `NodeEnum` 映射（静态方法）
  - `get_prefilter_enum()` — 返回预过滤模式的 `NodeEnum` 映射（静态方法）
  - `get_quality_enum()` — 返回质量等级的 `NodeEnum` 映射（静态方法）

## 核心函数

- `denoiserTypeToHumanReadable(DenoiserType type)` — 将降噪器类型枚举转换为人类可读字符串（"OptiX"、"OpenImageDenoise"）。
- `NODE_DEFINE(DenoiseParams)` — 通过 Node 系统注册 `DenoiseParams` 的所有套接字（Socket），包括 `use`、`type`、`start_sample`、`use_pass_albedo`、`use_pass_normal`、`temporally_stable`、`prefilter`、`quality` 等参数。

## 枚举类型

### DenoiserType
- `DENOISER_NONE = 0` — 无降噪器
- `DENOISER_OPTIX = 2` — NVIDIA OptiX 降噪器
- `DENOISER_OPENIMAGEDENOISE = 4` — Intel OpenImageDenoise 降噪器
- `DENOISER_ALL = ~0` — 所有降噪器（位掩码）

### DenoiserPrefilter
- `DENOISER_PREFILTER_NONE = 1` — 不预过滤，要求辅助通道无噪声
- `DENOISER_PREFILTER_FAST = 2` — 快速模式，同时降噪颜色和辅助通道
- `DENOISER_PREFILTER_ACCURATE = 3` — 精确模式，先预过滤辅助通道再降噪颜色

### DenoiserQuality
- `DENOISER_QUALITY_HIGH = 1` — 高质量
- `DENOISER_QUALITY_BALANCED = 2` — 平衡模式
- `DENOISER_QUALITY_FAST = 3` — 快速模式

## 依赖关系

- **内部头文件**: `graph/node.h`
- **外部库**: 无直接外部库依赖（OptiX 和 OIDN 的实际调用在其他文件中）
- **被引用**:
  - `src/device/device.h`
  - `src/scene/integrator.h`
  - `src/integrator/denoiser.h`
  - `src/integrator/denoiser_gpu.cpp`

## 实现细节 / 关键算法

- `DenoiseParams` 使用 Cycles 节点系统的 `SOCKET_BOOLEAN`、`SOCKET_ENUM`、`SOCKET_INT` 宏来声明参数套接字，使其可以通过统一的节点接口进行序列化、反序列化和参数传递。
- `NodeEnum` 采用懒初始化模式（通过静态局部变量），在首次调用时填充枚举映射，后续调用直接返回缓存结果。
- `DenoiserTypeMask` 是 `int` 的类型别名，用于位掩码操作以表示支持的降噪器组合。

## 关联文件

- `src/integrator/denoiser.h` / `denoiser.cpp` — 降噪器基类，使用 `DenoiseParams` 进行配置
- `src/integrator/denoiser_gpu.cpp` — GPU 降噪器实现
- `src/integrator/denoiser_oidn.cpp` — OpenImageDenoise 降噪器实现
- `src/scene/integrator.h` — 场景集成器，持有 `DenoiseParams` 实例
- `src/device/device.h` — 设备基类，引用降噪器类型用于能力描述
