# volume.h - 体积散射与消光闭包调度中心

## 概述

本文件是 Cycles 渲染器中体积（Volume）闭包系统的核心调度中心。它提供了体积消光（Extinction）的设置函数，以及体积相位函数（Phase Function）的统一求值、采样、比较和参数提取接口。通过 switch-case 调度机制，将具体的相位函数实现委托给各专用头文件（Henyey-Greenstein、Rayleigh、Draine、Fournier-Forand）。此外还包含体积颜色透射率计算和通道采样等工具函数。

## 类与结构体

本文件不定义新的结构体。各相位函数的结构体定义在各自的头文件中。

## 核心函数

### 体积消光

#### `volume_extinction_setup`
```cpp
ccl_device void volume_extinction_setup(ccl_private ShaderData *sd, Spectrum weight)
```
设置体积消光。若已有消光（`SD_EXTINCTION` 标志），则累加权重；否则设置标志并初始化消光值到 `closure_transparent_extinction`。

### 相位函数调度

#### `volume_phase_eval`
```cpp
ccl_device Spectrum volume_phase_eval(const ccl_private ShaderData *sd,
                                      const ccl_private ShaderVolumeClosure *svc,
                                      const float3 wo, ccl_private float *pdf)
```
根据闭包类型调度到对应的相位函数求值实现：
- `CLOSURE_VOLUME_FOURNIER_FORAND_ID` -> `volume_fournier_forand_eval`
- `CLOSURE_VOLUME_RAYLEIGH_ID` -> `volume_rayleigh_eval`
- `CLOSURE_VOLUME_DRAINE_ID` -> `volume_draine_eval`
- `CLOSURE_VOLUME_HENYEY_GREENSTEIN_ID` -> `volume_henyey_greenstein_eval`

#### `volume_phase_sample`
类似地调度到各相位函数的采样实现。

#### `volume_phase_equal`
比较两个体积闭包是否等价。不同类型始终不等；相同类型下比较各自的参数（如 g、alpha、c1/c2/c3 等）。

#### `volume_phase_get_g`
提取相位函数的各向异性参数 g（Henyey-Greenstein 等效），用于体积引导（Volume Guiding）。各类型的近似：
- Fournier-Forand: 返回 1.0（TODO，待改进）
- Rayleigh: 返回 0.0（近似各向同性）
- Draine: 返回其 g 参数
- Henyey-Greenstein: 返回其 g 参数

### 体积工具函数

#### `volume_color_transmittance`
```cpp
ccl_device Spectrum volume_color_transmittance(Spectrum sigma, float t)
```
计算体积颜色透射率：`exp(-sigma * t)`，即 Beer-Lambert 定律。

#### `volume_channel_get`
从光谱值中获取指定通道的值。

#### `volume_sample_channel_pdf`
```cpp
ccl_device_inline Spectrum volume_sample_channel_pdf(Spectrum albedo, Spectrum throughput)
```
按通量和单散射反照率的乘积比例计算各光谱通道的采样概率。基于 Chiang 等人 2016 SIGGRAPH 论文的方法，显著降低多次弹射体积散射的噪声。

#### `volume_sample_channel`
根据 `volume_sample_channel_pdf` 计算的概率选择一个光谱通道，并重用随机数。

## 依赖关系
- **内部头文件**:
  - `kernel/types.h` — 内核类型定义
  - `kernel/closure/volume_draine.h` — Draine 相位函数
  - `kernel/closure/volume_fournier_forand.h` — Fournier-Forand 相位函数
  - `kernel/closure/volume_henyey_greenstein.h` — Henyey-Greenstein 相位函数
  - `kernel/closure/volume_rayleigh.h` — Rayleigh 相位函数
- **被引用**:
  - `src/kernel/svm/closure.h` — SVM 闭包节点
  - `src/kernel/integrator/subsurface_random_walk.h` — 随机游走次表面散射
  - `src/kernel/integrator/volume_shader.h` — 体积着色器积分
  - `src/kernel/osl/closures_setup.h` — OSL 闭包初始化
  - `src/kernel/integrator/shade_volume.h` — 体积着色

## 实现细节 / 关键算法

### 体积通道采样策略
`volume_sample_channel_pdf` 使用 `|throughput * albedo|` 作为各通道权重。这意味着通量高且反照率大的通道被采样的概率更高，有效减少了高散射次数时的方差。当所有权重为零时（避免除零），回退到均匀采样。

### 体积吞吐量截止
宏 `VOLUME_THROUGHPUT_EPSILON`（1e-6）定义了体积通量的最小阈值。低于此值的路径将被忽略，以避免不必要的计算和精度问题。

### 相位函数等价性判断
`volume_phase_equal` 用于体积着色器优化中合并等效闭包。各类型比较方式不同：
- Rayleigh 无参数，同类型即等价
- Henyey-Greenstein 比较 g
- Draine 比较 g 和 alpha
- Fournier-Forand 比较预计算系数 c1、c2、c3

## 关联文件
- `src/kernel/closure/volume_henyey_greenstein.h` — HG 相位函数实现
- `src/kernel/closure/volume_rayleigh.h` — Rayleigh 相位函数实现
- `src/kernel/closure/volume_draine.h` — Draine 相位函数实现
- `src/kernel/closure/volume_fournier_forand.h` — FF 相位函数实现
- `src/kernel/integrator/shade_volume.h` — 体积积分器
- `src/kernel/integrator/volume_shader.h` — 体积着色器求值
