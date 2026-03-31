# kernel.h / kernel.cpp - CPU 内核函数集合的定义与注册

## 概述

本文件定义了 `CPUKernels` 类，它是 Cycles 渲染引擎中所有 CPU 端内核函数的集中注册表。`kernel.h` 声明了各类内核函数的类型别名和成员变量，`kernel.cpp` 通过宏机制在构造函数中完成所有内核函数的注册，为每个内核绑定默认实现和 AVX2 优化实现。

## 类与结构体

### CPUKernels

- **继承**: 无（独立类）
- **功能**: 作为 CPU 端全部渲染内核函数的容器，统一管理积分器、着色器求值、自适应采样、体积引导、Cryptomatte 后处理和胶片转换等内核。每个内核成员均为 `CPUKernelFunction` 模板实例，自动根据 CPU 指令集选择最优实现。

#### 类型别名

| 别名 | 函数签名 | 用途 |
|------|----------|------|
| `IntegratorFunction` | `void (*)(const ThreadKernelGlobalsCPU*, IntegratorStateCPU*)` | 通用积分器内核 |
| `IntegratorShadeFunction` | `void (*)(const ThreadKernelGlobalsCPU*, IntegratorStateCPU*, float*)` | 带渲染缓冲区的着色积分器内核 |
| `IntegratorInitFunction` | `bool (*)(const ThreadKernelGlobalsCPU*, IntegratorStateCPU*, KernelWorkTile*, float*)` | 积分器初始化内核，返回是否成功 |
| `ShaderEvalFunction` | `void (*)(const ThreadKernelGlobalsCPU*, const KernelShaderEvalInput*, float*, int)` | 着色器求值内核 |
| `AdaptiveSamplingConvergenceCheckFunction` | `bool (*)(const ThreadKernelGlobalsCPU*, float*, int, int, float, int, int, int)` | 自适应采样收敛检查内核 |
| `FilterXFunction` | `void (*)(const ThreadKernelGlobalsCPU*, float*, int, int, int, int, int)` | X 方向滤波内核 |
| `FilterYFunction` | `void (*)(const ThreadKernelGlobalsCPU*, float*, int, int, int, int, int)` | Y 方向滤波内核 |
| `CryptomattePostprocessFunction` | `void (*)(const ThreadKernelGlobalsCPU*, float*, int)` | Cryptomatte 后处理内核 |
| `FilmConvertFunction` | `void (*)(const KernelFilmConvert*, const float*, float*, int, int, int)` | 胶片转换内核（浮点输出） |
| `FilmConvertHalfRGBAFunction` | `void (*)(const KernelFilmConvert*, const float*, half4*, int, int)` | 胶片转换内核（半精度 RGBA 输出） |

#### 关键成员 - 积分器内核

| 成员 | 说明 |
|------|------|
| `integrator_init_from_camera` | 从相机初始化路径追踪积分器 |
| `integrator_init_from_bake` | 从烘焙任务初始化积分器 |
| `integrator_megakernel` | 巨内核模式的路径追踪积分器（包含完整的着色流程） |

#### 关键成员 - 着色器求值内核

| 成员 | 说明 |
|------|------|
| `shader_eval_displace` | 位移着色器求值 |
| `shader_eval_background` | 背景着色器求值 |
| `shader_eval_curve_shadow_transparency` | 曲线阴影透明度着色器求值 |
| `shader_eval_volume_density` | 体积密度着色器求值 |

#### 关键成员 - 自适应采样内核

| 成员 | 说明 |
|------|------|
| `adaptive_sampling_convergence_check` | 检查像素是否已收敛 |
| `adaptive_sampling_filter_x` | X 方向自适应采样滤波 |
| `adaptive_sampling_filter_y` | Y 方向自适应采样滤波 |

#### 关键成员 - 体积散射概率引导内核

| 成员 | 说明 |
|------|------|
| `volume_guiding_filter_x` | X 方向体积引导滤波 |
| `volume_guiding_filter_y` | Y 方向体积引导滤波 |

#### 关键成员 - 其他内核

| 成员 | 说明 |
|------|------|
| `cryptomatte_postprocess` | Cryptomatte 通道后处理 |
| `film_convert_*` | 胶片转换系列内核，通过 `KERNEL_FILM_CONVERT_FUNCTION` 宏批量定义，覆盖 depth、mist、volume_majorant、sample_count、float、light_path、rgbe、float3、motion、cryptomatte、shadow_catcher、shadow_catcher_matte_with_shadow、combined、float4 等通道类型 |

#### 关键方法

| 方法 | 说明 |
|------|------|
| `CPUKernels()` | 构造函数。通过 `REGISTER_KERNEL` 和 `REGISTER_KERNEL_FILM_CONVERT` 宏注册全部内核函数 |

## 核心函数

### 宏定义（kernel.cpp）

```cpp
#define KERNEL_FUNCTIONS(name) KERNEL_NAME_EVAL(cpu, name), KERNEL_NAME_EVAL(cpu_avx2, name)
#define REGISTER_KERNEL(name) name(KERNEL_FUNCTIONS(name))
#define REGISTER_KERNEL_FILM_CONVERT(name) \
  film_convert_##name(KERNEL_FUNCTIONS(film_convert_##name)), \
      film_convert_half_rgba_##name(KERNEL_FUNCTIONS(film_convert_half_rgba_##name))
```

- `KERNEL_FUNCTIONS(name)` — 展开为默认 CPU 实现和 AVX2 优化实现的函数名对。
- `REGISTER_KERNEL(name)` — 用于在构造函数初始化列表中注册单个内核（传入两个函数指针给 `CPUKernelFunction` 构造函数）。
- `REGISTER_KERNEL_FILM_CONVERT(name)` — 注册一对胶片转换内核（浮点版 + 半精度 RGBA 版）。

## 依赖关系

### 内部头文件
- `device/cpu/kernel_function.h` — `CPUKernelFunction` 模板类，内核函数的微架构分发包装器
- `util/half.h` — `half4` 半精度浮点类型
- `kernel/device/cpu/kernel.h` — CPU 端内核函数的实际声明（`KERNEL_NAME_EVAL` 宏展开后的目标函数）

### 被引用
- `src/device/cpu/device_impl.h` / `device_impl.cpp` — `CPUDevice` 中引用内核集合
- `src/device/device.cpp` — `Device::get_cpu_kernels()` 静态方法中实例化 `CPUKernels`
- `src/device/device.h` — 声明 `get_cpu_kernels()` 返回 `const CPUKernels&`
- `src/integrator/path_trace_work_cpu.cpp` — CPU 路径追踪工作中直接调用内核函数
- `src/integrator/pass_accessor_cpu.h` / `pass_accessor_cpu.cpp` — CPU 通道访问器使用胶片转换内核
- `src/integrator/shader_eval.cpp` — 着色器求值中使用着色器内核

## 实现细节 / 关键算法

### 内核注册机制

`CPUKernels` 的构造函数使用 C++ 初始化列表，配合宏展开，一次性注册全部内核。每个 `CPUKernelFunction` 接收两个函数指针（默认实现和 AVX2 实现），在运行时由 `CPUKernelFunction` 内部根据 CPU 能力自动选择最优版本。

### 胶片转换宏批量定义

`KERNEL_FILM_CONVERT_FUNCTION(name)` 宏在头文件中为每种通道类型同时定义两个成员：
- `film_convert_##name` — 浮点输出版本
- `film_convert_half_rgba_##name` — 半精度 RGBA 输出版本

共覆盖 14 种通道类型（depth、mist、volume_majorant、sample_count、float、light_path、rgbe、float3、motion、cryptomatte、shadow_catcher、shadow_catcher_matte_with_shadow、combined、float4），即 28 个胶片转换内核函数。

### 全局单例

`CPUKernels` 在 `Device::get_cpu_kernels()` 中以静态局部变量方式创建，整个进程生命周期内只有一个实例，避免重复初始化开销。

## 关联文件

- `src/device/cpu/kernel_function.h` — `CPUKernelFunction` 模板，内核函数的运行时分发机制
- `src/kernel/device/cpu/kernel.h` — 底层 CPU 内核函数声明
- `src/device/cpu/device_impl.h` — `CPUDevice` 类，内核的使用方
- `src/integrator/path_trace_work_cpu.cpp` — 路径追踪工作调度，内核函数的主要调用者
