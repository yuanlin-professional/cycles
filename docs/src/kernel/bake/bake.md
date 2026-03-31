# bake.h - 着色器烘焙求值内核函数

## 概述

`bake.h` 是 Cycles 渲染器烘焙（Bake）子系统的核心头文件，提供了四种离线着色器求值内核函数。这些函数不参与实时路径追踪流程，而是在预计算阶段对置换、背景、曲线阴影透明度和体积密度进行批量求值，将结果写入输出缓冲区供后续渲染管线使用。该文件的设计遵循"输入数组 + 偏移索引 -> 输出数组"的 GPU 友好并行求值模式，每个函数处理单个工作项，由设备层（CPU/GPU/OptiX）负责调度并行执行。

## 类与结构体

本文件不定义新的类或结构体，但核心依赖以下数据结构：

### KernelShaderEvalInput
- **定义位置**: `kernel/types.h`
- **功能**: 着色器求值的输入参数包，16 字节对齐
- **关键成员**:
  - `object` — 物体索引（int）
  - `prim` — 图元索引（int）
  - `u, v` — 参数化坐标（float），根据求值类型含义不同（UV 坐标、等距柱状投影坐标、空间坐标等）

### ShaderData
- **定义位置**: `kernel/osl/services.h`（声明）、`kernel/geom/shader_data.h`（初始化辅助函数）
- **功能**: 着色器求值上下文，存储几何信息、闭包列表、发射/消光系数等

## 枚举与常量

本文件使用但不定义以下关键枚举/常量（均定义于 `kernel/types.h`）：

| 名称 | 值 | 说明 |
|------|---|------|
| `PATH_RAY_IMPORTANCE_BAKE` | `1U << 8U` | 烘焙专用路径标志，在背景求值中用于忽略太阳引导贴图中的太阳盘 |
| `PATH_RAY_EMISSION` | — | 发射光线标志，与上述标志组合用于背景求值 |
| `PATH_RAY_SHADOW` | — | 阴影光线标志，用于曲线透明度求值 |
| `PATH_RAY_CAMERA` | — | 相机光线标志，用于体积密度求值 |
| `SD_IS_VOLUME_SHADER_EVAL` | `1 << 8` | ShaderData 标志，标记当前为体积着色器求值模式 |
| `PRNG_BAKE_VOLUME_DENSITY_EVAL` | `0` | 体积密度烘焙专用的 Sobol-Burley 采样维度偏移 |
| `INTEGRATOR_STATE_NULL` | — | 空积分器状态，因烘焙不走完整积分器路径 |

## 核心函数

### kernel_displace_evaluate()
- **签名**: `ccl_device void kernel_displace_evaluate(KernelGlobals kg, const ccl_global KernelShaderEvalInput *input, ccl_global float *output, const int offset)`
- **功能**: 对单个图元执行置换着色器求值。根据输入的物体/图元/UV 信息初始化 `ShaderData`，调用 `displacement_shader_eval()` 计算置换后的位置偏移向量 D，再通过 `object_inverse_dir_transform()` 变换回物体空间。最终将 D 的三个分量累加写入输出缓冲区（`offset * 3` 起始）。
- **安全机制**: 在调试模式（`__KERNEL_DEBUG_NAN__`）下检测非有限值并断言；始终调用 `ensure_finite()` 确保输出有限，防止层次包围体(BVH)退化。

### kernel_background_evaluate()
- **签名**: `ccl_device void kernel_background_evaluate(KernelGlobals kg, const ccl_global KernelShaderEvalInput *input, ccl_global float *output, const int offset)`
- **功能**: 对单个方向执行背景/环境着色器求值。将输入的 (u, v) 通过 `equirectangular_to_direction()` 转换为球面方向向量，构造从原点出发的光线并初始化背景 `ShaderData`，调用 `surface_shader_eval()` 求值后提取背景颜色。
- **路径标志**: 使用 `PATH_RAY_EMISSION | PATH_RAY_IMPORTANCE_BAKE` 组合，前者标记发射求值，后者在 SVM Sky 节点中控制是否忽略太阳盘（避免重复计算太阳引导）。
- **着色器特征掩码**: 排除了 `KERNEL_FEATURE_NODE_RAYTRACE` 和 `KERNEL_FEATURE_NODE_LIGHT_PATH`，因为烘焙环境中无法执行光线追踪节点和光路查询节点。
- **输出**: 将 `Spectrum` 转换为 RGB 后写入输出缓冲区（`offset * 3` 起始）。

### kernel_curve_shadow_transparency_evaluate()
- **签名**: `ccl_device void kernel_curve_shadow_transparency_evaluate(KernelGlobals kg, const ccl_global KernelShaderEvalInput *input, ccl_global float *output, const int offset)`
- **功能**: 对单条曲线（毛发）图元求值阴影透明度。仅在 `__HAIR__` 编译宏启用时生效。通过 `shader_setup_from_curve()` 初始化曲线着色数据，以 `PATH_RAY_SHADOW` 标志调用表面着色器，取透明度通道的平均值并 clamp 到 [0, 1] 范围。
- **输出**: 单个标量值写入 `output[offset]`。

### kernel_volume_density_evaluate()
- **签名**: `ccl_device void kernel_volume_density_evaluate(KernelGlobals kg, ccl_global const KernelShaderEvalInput *input, ccl_global float *output, const int offset)`
- **功能**: 对单个体素估算体积密度的极值范围（最小值与最大值）。仅在 `__VOLUME__` 编译宏启用时生效。该函数是体积八叉树（octree）构建流程的核心，为自适应体积步进提供密度边界信息。
- **输入格式**: 每个工作项占用输入数组中的两个连续条目（`offset * 2` 和 `offset * 2 + 1`）：
  - 第一个条目：体素中心位置（编码在 prim/u/v 中）
  - 第二个条目：体积着色器 ID（object 字段）、体素尺寸（编码在 prim/u/v 中）；若 `object == SHADER_NONE` 则跳过
- **采样策略**: 对异质体积在体素内取 16 个 Sobol-Burley 蓝噪声采样点，对均质体积仅取 1 个采样点。在每个采样点求值消光系数和发射强度，取两者最大分量更新极值。
- **坐标变换**: 若物体变换未预烘焙（`SD_OBJECT_TRANSFORM_APPLIED` 标志未设置），则将采样点从物体空间变换到世界空间。
- **输出**: 两个标量值写入 `output[offset * 2]`（最小密度）和 `output[offset * 2 + 1]`（最大密度），均除以物体体积密度缩放因子。

## 依赖关系

### 内部头文件
- `kernel/globals.h` — 内核全局数据定义（`KernelGlobals`、`kernel_data_fetch` 等）
- `kernel/camera/projection.h` — 相机投影工具函数（`equirectangular_to_direction()`）
- `kernel/integrator/displacement_shader.h` — 置换着色器求值（`displacement_shader_eval()`）
- `kernel/integrator/surface_shader.h` — 表面着色器求值（`surface_shader_eval()`、`surface_shader_background()`、`surface_shader_transparency()`）
- `kernel/integrator/volume_shader.h` — 体积着色器求值（`volume_shader_eval_entry()`、`volume_is_homogeneous()`）
- `kernel/geom/object.h` — 物体几何变换函数（`object_inverse_dir_transform()`、`object_fetch_transform()`、`object_volume_density()`）
- `kernel/geom/shader_data.h` — `ShaderData` 初始化函数（`shader_setup_from_displace()`、`shader_setup_from_background()`、`shader_setup_from_curve()`、`shader_setup_from_volume()`）
- `kernel/util/colorspace.h` — 色彩空间转换工具（`spectrum_to_rgb()`）

### 外部库
无直接外部库依赖。

### 被引用
- `src/kernel/device/gpu/kernel.h` — GPU 通用内核调度层，将四个烘焙函数包装为 GPU kernel 入口
- `src/kernel/device/cpu/kernel_arch_impl.h` — CPU 架构特化内核实现，调用烘焙函数的 CPU 路径
- `src/kernel/device/optix/kernel_osl.cu` — OptiX + OSL 后端，在 OptiX launch 回调中调用烘焙函数
- `src/kernel/CMakeLists.txt` — 构建系统中注册本文件

## 实现细节 / 关键算法

### 并行求值模型
所有四个函数共享相同的接口模式：`(KernelGlobals, input[], output[], offset)`。`offset` 是当前工作项的全局索引，由设备调度层计算（GPU 为线程索引 + 偏移，CPU 为循环变量）。这种设计使得同一份代码可以无修改地在 CPU、CUDA、HIP、OptiX、Metal 等后端运行。

### 体积密度极值估算
`kernel_volume_density_evaluate()` 的核心算法为蒙特卡洛极值估算：在体素内随机采样多个点，对每个点求值消光和发射系数，取所有采样中的全局最小值和最大值。该极值对用于构建体积八叉树，指导路径追踪中的自适应步进——密度为零的区域可以跳过，密度变化剧烈的区域需要更细的步长。采样使用 Sobol-Burley 序列保证低差异性，并通过蓝噪声索引降低结构化噪声。

### NaN/Inf 安全保护
置换和背景求值函数均包含 `ensure_finite()` 调用。这是因为非有限值（NaN/Inf）会导致层次包围体(BVH)退化（包围盒无限大）或路径追踪内核中的数值不稳定。调试模式下还会触发断言以便开发者定位问题。

### 着色器特征掩码裁剪
背景和曲线透明度求值在调用 `surface_shader_eval()` 时使用模板参数排除了光线追踪节点（`KERNEL_FEATURE_NODE_RAYTRACE`）和光路查询节点（`KERNEL_FEATURE_NODE_LIGHT_PATH`），因为烘焙上下文中没有完整的积分器状态来支持这些节点的执行。

## 关联文件

| 文件 | 关系 |
|------|------|
| `src/kernel/types.h` | 定义 `KernelShaderEvalInput`、路径标志、着色器标志等 |
| `src/kernel/device/gpu/kernel.h` | GPU 内核入口，调度本文件中的烘焙函数 |
| `src/kernel/device/cpu/kernel_arch_impl.h` | CPU 内核实现，调用本文件中的烘焙函数 |
| `src/kernel/device/optix/kernel_osl.cu` | OptiX OSL 后端调用入口 |
| `src/kernel/integrator/displacement_shader.h` | 置换着色器求值实现 |
| `src/kernel/integrator/surface_shader.h` | 表面着色器求值实现 |
| `src/kernel/integrator/volume_shader.h` | 体积着色器求值实现 |
| `src/kernel/geom/shader_data.h` | ShaderData 初始化辅助函数 |
| `src/kernel/camera/projection.h` | 等距柱状投影方向转换 |
