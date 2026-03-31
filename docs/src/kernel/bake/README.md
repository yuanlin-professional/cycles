# kernel/bake - 烘焙

## 概述

`kernel/bake/` 实现了 Cycles 渲染器的着色器评估烘焙功能。烘焙是一种将着色器计算结果（位移、背景环境光、曲线阴影透明度）预计算并存储的技术，用于加速后续渲染或为其他系统提供预计算数据。该模块包含三个独立的烘焙评估内核：位移烘焙、背景烘焙和曲线阴影透明度烘焙。

与路径追踪渲染不同，烘焙内核不执行光线追踪，而是直接在指定的几何表面点上设置着色数据（`ShaderData`），评估相应的着色器，并将结果写入输出缓冲区。

## 目录结构

| 文件名 | 类型 | 说明 |
|--------|------|------|
| `bake.h` | 头文件 | 烘焙评估内核实现：位移着色器评估、背景着色器评估、曲线阴影透明度着色器评估 |

## 核心类与数据结构

### KernelShaderEvalInput
烘焙输入结构体，为每个评估点提供：
- `object` - 物体索引
- `prim` - 图元索引
- `u`, `v` - 表面参数坐标（对于背景烘焙，u/v 表示等距柱形投影坐标）

### ShaderData
着色器数据结构体（定义于 `kernel/types.h`），在烘焙中被设置为包含：
- 表面位置 `P`
- 法线 `N`、`Ng`
- UV 坐标
- 物体和图元引用
- 着色器评估结果

## 内核函数入口

| 函数 | 文件 | 说明 |
|------|------|------|
| `kernel_displace_evaluate()` | `bake.h` | **位移烘焙**：在指定图元的 (u, v) 位置设置着色数据，评估位移着色器，输出位移向量 (dx, dy, dz)。结果经过逆物体变换转回物体空间，并确保有限值以防止 BVH 退化 |
| `kernel_background_evaluate()` | `bake.h` | **背景烘焙**：将 (u, v) 坐标转换为等距柱形投影方向，评估背景着色器，输出 RGB 颜色值。使用 `PATH_RAY_IMPORTANCE_BAKE` 标志忽略背景贴图中的太阳光源 |
| `kernel_curve_shadow_transparency_evaluate()` | `bake.h` | **曲线阴影透明度烘焙**：评估曲线几何体的阴影透明度着色器，用于预计算毛发/曲线的半透明阴影 |

## GPU 兼容性

- 所有函数使用 `ccl_device` 修饰，在 GPU 和 CPU 上均可编译
- 输入使用 `ccl_global` 指针访问全局设备内存中的 `KernelShaderEvalInput` 数组
- 输出使用 `ccl_global float*` 指针写入设备内存缓冲区
- 包含 `__KERNEL_DEBUG_NAN__` 条件编译的 NaN 检测，帮助调试非有限值问题
- 使用 `ensure_finite()` 确保输出值有限，避免后续 BVH 构建或路径追踪中的数值问题

## 依赖关系

### 上游依赖（本模块依赖）
- `kernel/globals.h` - 内核全局数据访问
- `kernel/camera/projection.h` - `equirectangular_to_direction()`（背景烘焙的方向计算）
- `kernel/integrator/displacement_shader.h` - `displacement_shader_eval()`（位移着色器评估）
- `kernel/integrator/surface_shader.h` - `surface_shader_eval()`、`surface_shader_background()`（表面着色器评估）
- `kernel/integrator/volume_shader.h` - 体积着色器相关
- `kernel/geom/object.h` - `object_inverse_dir_transform()`（物体空间变换）
- `kernel/geom/shader_data.h` - `shader_setup_from_displace()`、`shader_setup_from_background()`
- `kernel/util/colorspace.h` - `spectrum_to_rgb()`（颜色转换）

### 下游依赖（依赖本模块）
- `kernel/device/*/kernel.h` - 各设备后端调用烘焙内核（通过 `KERNEL_FEATURE_BAKING` 特性标志控制）
- `src/session/` - 渲染会话通过设备接口调度烘焙任务

## 关键算法与实现细节

### 位移烘焙流程
1. 从输入数组读取 `(object, prim, u, v)` 参数
2. 调用 `shader_setup_from_displace()` 在指定位置初始化 `ShaderData`
3. 记录原始位置 `P`
4. 调用 `displacement_shader_eval()` 评估位移着色器（修改 `sd.P`）
5. 计算位移向量 `D = sd.P - P`
6. 通过 `object_inverse_dir_transform()` 将位移从世界空间变换回物体空间
7. 确保结果有限（`ensure_finite`），防止 BVH 退化
8. 累加写入输出缓冲区（使用 `+=` 支持多次评估叠加）

### 背景烘焙流程
1. 将 `(u, v)` 坐标通过 `equirectangular_to_direction()` 转换为世界空间方向
2. 从方向设置背景着色数据（`shader_setup_from_background()`）
3. 使用 `PATH_RAY_EMISSION | PATH_RAY_IMPORTANCE_BAKE` 路径标志评估着色器
4. 通过 `surface_shader_background()` 获取背景颜色
5. 转换为 RGB 并写入输出

## 参见

- `src/kernel/integrator/displacement_shader.h` - 位移着色器评估实现
- `src/kernel/integrator/surface_shader.h` - 表面着色器评估实现
- `src/kernel/camera/projection.h` - 等距柱形投影变换
- `src/kernel/geom/shader_data.h` - 着色数据初始化
