# kernel_arch.h - CPU 内核架构函数声明模板

## 概述

本文件是一个可重复包含的模板头文件（不使用 `#pragma once`），通过预定义的 `KERNEL_ARCH` 宏为不同 CPU 指令集架构生成对应的内核函数声明。每次包含时 `KERNEL_FUNCTION_FULL_NAME(name)` 会展开为 `kernel_<KERNEL_ARCH>_<name>` 形式，从而为同一函数生成不同架构的版本声明。

## 核心函数/宏定义

### 积分器函数声明宏
- **`KERNEL_INTEGRATOR_FUNCTION(name)`**: 声明积分器基础函数，签名为 `void kernel_<arch>_integrator_<name>(kg, state)`
- **`KERNEL_INTEGRATOR_SHADE_FUNCTION(name)`**: 声明着色函数，增加 `render_buffer` 参数
- **`KERNEL_INTEGRATOR_INIT_FUNCTION(name)`**: 声明初始化函数，增加 `tile` 和 `render_buffer` 参数，返回 `bool`

### 积分器函数声明
- `integrator_init_from_camera` — 从相机初始化积分器状态
- `integrator_init_from_bake` — 从烘焙初始化积分器状态
- `integrator_megakernel` — 大内核模式的主循环

### 胶片转换函数 (`KERNEL_FILM_CONVERT_FUNCTION`)
为每种通道类型声明两个函数：`film_convert_<name>` 和 `film_convert_half_rgba_<name>`：
- 单通道浮点: `depth`, `mist`, `sample_count`, `volume_majorant`, `float`
- 三通道: `light_path`, `rgbe`, `float3`
- 四通道: `motion`, `cryptomatte`, `shadow_catcher`, `shadow_catcher_matte_with_shadow`, `combined`, `float4`

### 着色器评估函数
- `shader_eval_background` — 背景着色器评估
- `shader_eval_displace` — 位移着色器评估
- `shader_eval_curve_shadow_transparency` — 曲线阴影透明度评估
- `shader_eval_volume_density` — 体积密度着色器评估

### 自适应采样函数
- `adaptive_sampling_convergence_check` — 收敛检查（返回 `bool`）
- `adaptive_sampling_filter_x` — X 方向自适应采样滤波
- `adaptive_sampling_filter_y` — Y 方向自适应采样滤波

### Cryptomatte 后处理
- `cryptomatte_postprocess` — Cryptomatte 通道后处理

### 体积散射概率引导
- `volume_guiding_filter_x` — X 方向体积引导滤波
- `volume_guiding_filter_y` — Y 方向体积引导滤波

## 依赖关系

- **内部头文件**: 依赖调用者预先定义 `KERNEL_ARCH` 宏和 `KERNEL_FUNCTION_FULL_NAME` 宏（来自 `kernel.h`）
- **被引用**: `src/kernel/device/cpu/kernel.h`（被包含两次，分别用于 `cpu` 和 `cpu_avx2` 架构）

## 实现细节 / 关键算法

### 可重复包含设计
本文件故意不使用 `#pragma once` 或头文件保护宏，因为它需要被同一个文件（`kernel.h`）包含多次，每次 `KERNEL_ARCH` 值不同：
1. 第一次：`KERNEL_ARCH=cpu` -> 生成 `kernel_cpu_integrator_init_from_camera` 等
2. 第二次：`KERNEL_ARCH=cpu_avx2` -> 生成 `kernel_cpu_avx2_integrator_init_from_camera` 等

文件末尾 `#undef KERNEL_ARCH` 清理宏定义，为下次包含做准备。

## 关联文件

- `src/kernel/device/cpu/kernel.h` — 包含本文件的父文件
- `src/kernel/device/cpu/kernel_arch_impl.h` — 对应的函数实现模板
