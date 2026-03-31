# kernel.h - GPU 通用内核入口与调度中心

## 概述

本文件是 Cycles 渲染器 GPU 设备内核的核心入口文件，定义了所有在 GPU 上执行的计算内核函数。它汇聚了路径追踪积分器的各个阶段（初始化、光线求交、着色）、自适应采样、胶片转换、着色器评估、降噪和阴影捕捉器等全部 GPU 内核。各后端（CUDA、HIP、Metal、OneAPI）通过包含此文件来获得统一的内核实现。

## 核心函数

### 积分器内核

#### integrator_reset()
- **签名**: `ccl_gpu_kernel_signature(integrator_reset, const int num_states)`
- **功能**: 重置所有积分器路径状态，将主路径和阴影路径的队列内核标记清零。

#### integrator_init_from_camera()
- **签名**: `ccl_gpu_kernel_signature(integrator_init_from_camera, ccl_global KernelWorkTile *tiles, const int num_tiles, ccl_global float *render_buffer, const int max_tile_work_size)`
- **功能**: 从相机参数初始化路径追踪状态。根据全局工作索引定位到具体的图块（tile）、像素坐标和采样编号，调用 `get_work_pixel()` 进行工作分配。

#### integrator_init_from_bake()
- **签名**: `ccl_gpu_kernel_signature(integrator_init_from_bake, ccl_global KernelWorkTile *tiles, const int num_tiles, ccl_global float *render_buffer, const int max_tile_work_size)`
- **功能**: 从烘焙（bake）参数初始化路径追踪状态，流程与相机初始化类似，但数据源为烘焙输入。

### 光线求交内核

#### integrator_intersect_closest()
- **签名**: `ccl_gpu_kernel_signature(integrator_intersect_closest, const ccl_global int *path_index_array, ccl_global float *render_buffer, const int work_size)`
- **功能**: 对主路径执行最近交点光线求交。

#### integrator_intersect_shadow()
- **签名**: `ccl_gpu_kernel_signature(integrator_intersect_shadow, const ccl_global int *path_index_array, const int work_size)`
- **功能**: 对阴影光线执行求交测试。

#### integrator_intersect_subsurface()
- **签名**: `ccl_gpu_kernel_signature(integrator_intersect_subsurface, const ccl_global int *path_index_array, const int work_size)`
- **功能**: 对次表面散射光线执行求交。

#### integrator_intersect_volume_stack()
- **签名**: `ccl_gpu_kernel_signature(integrator_intersect_volume_stack, const ccl_global int *path_index_array, const int work_size)`
- **功能**: 构建体积堆栈的求交操作。

#### integrator_intersect_dedicated_light()
- **签名**: `ccl_gpu_kernel_signature(integrator_intersect_dedicated_light, const ccl_global int *path_index_array, const int work_size)`
- **功能**: 对专用光源光线执行求交。

### 着色内核

#### integrator_shade_background()
- **签名**: `ccl_gpu_kernel_signature(integrator_shade_background, const ccl_global int *path_index_array, ccl_global float *render_buffer, const int work_size)`
- **功能**: 背景着色，处理未命中任何物体的光线。

#### integrator_shade_light()
- **签名**: 同上模式
- **功能**: 光源着色。

#### integrator_shade_shadow()
- **签名**: 同上模式
- **功能**: 阴影着色。

#### integrator_shade_surface()
- **签名**: 同上模式
- **功能**: 表面着色，处理漫反射、光泽等 BSDF。

#### integrator_shade_surface_raytrace()
- **签名**: 同上模式
- **功能**: 带有光线追踪功能的表面着色（支持环境光遮蔽和斜面节点）。包含 Metal 平台的 `__dummy_constant` 变通方案以修复 AO/Bevel 节点异常。

#### integrator_shade_surface_mnee()
- **签名**: 同上模式
- **功能**: 使用 MNEE（Manifold Next Event Estimation）方法的表面着色，优化焦散效果的计算。

#### integrator_shade_volume() / integrator_shade_volume_ray_marching()
- **签名**: 同上模式
- **功能**: 体积着色和体积光线步进（ray marching）。

#### integrator_shade_dedicated_light()
- **签名**: 同上模式
- **功能**: 专用光源着色。

### 路径状态管理内核

#### integrator_queued_paths_array()
- **签名**: `ccl_gpu_kernel_signature(integrator_queued_paths_array, const int num_states, ccl_global int *indices, ccl_global int *num_indices, const int kernel_index)`
- **功能**: 构建匹配指定内核索引的队列路径索引数组。使用 `gpu_parallel_active_index_array` 实现并行压缩。

#### integrator_active_paths_array()
- **签名**: `ccl_gpu_kernel_signature(integrator_active_paths_array, const int num_states, ccl_global int *indices, ccl_global int *num_indices)`
- **功能**: 构建所有活跃路径（queued_kernel != 0）的索引数组。

#### integrator_terminated_paths_array()
- **签名**: 类似 active_paths，增加 `indices_offset` 参数
- **功能**: 构建所有已终止路径的索引数组。

#### integrator_sorted_paths_array()
- **签名**: `ccl_gpu_kernel_signature(integrator_sorted_paths_array, const int num_states, const int num_states_limit, ccl_global int *indices, ccl_global int *num_indices, ccl_global int *key_counter, ccl_global int *key_prefix_sum, const int kernel_index)`
- **功能**: 构建按着色器排序键排序的活跃路径索引数组。利用 `gpu_parallel_sorted_index_array` 实现。

#### integrator_sort_bucket_pass() / integrator_sort_write_pass()
- **功能**: 基于本地原子操作的两趟排序算法。bucket pass 计算分区内各桶的大小，write pass 将排序后的索引写入全局数组。使用 `__KERNEL_LOCAL_ATOMIC_SORT__` 条件编译。

#### integrator_compact_paths_array() / integrator_compact_shadow_paths_array()
- **功能**: 压缩路径数组，收集需要重新排列的活跃路径索引。

#### integrator_compact_states() / integrator_compact_shadow_states()
- **功能**: 执行实际的路径状态搬迁（从分散的位置移动到紧凑的连续位置）。

#### prefix_sum()
- **签名**: `ccl_gpu_kernel_signature(prefix_sum, ccl_global int *counter, ccl_global int *prefix_sum, const int num_values)`
- **功能**: 对计数器数组执行前缀和计算，GPU 内核包装层。

### 自适应采样内核

#### adaptive_sampling_convergence_check()
- **功能**: 检查每个像素是否已收敛。使用波前投票（`ccl_gpu_ballot`）统计未收敛像素数量，高效利用 warp 级并行。

#### adaptive_sampling_filter_x() / adaptive_sampling_filter_y()
- **功能**: 自适应采样的水平和垂直滤波通道。

### 胶片转换内核

#### KERNEL_FILM_CONVERT_VARIANT 宏
- **功能**: 宏生成一系列胶片通道转换内核，包括 float 和 half_rgba 两个版本。覆盖的变体有：depth、mist、volume_majorant、sample_count、float（1通道），light_path、rgbe、float3（3通道），motion、cryptomatte、shadow_catcher、combined、float4（4通道）等。

#### kernel_gpu_film_convert_half_write()
- **签名**: `ccl_device_inline void kernel_gpu_film_convert_half_write(ccl_global uchar4 *rgba, const int rgba_offset, const int rgba_stride, const int x, const int y, const half4 half_pixel)`
- **功能**: 将半精度像素数据写入 RGBA 缓冲区。包含针对 HIP 平台的逐分量写入变通方案（修复 half float 显示问题，参见 #92972）。

### 着色器评估内核

#### shader_eval_displace()
- **功能**: 置换着色器评估。

#### shader_eval_background()
- **功能**: 背景着色器评估。

#### shader_eval_curve_shadow_transparency()
- **功能**: 曲线阴影透明度着色器评估。

#### shader_eval_volume_density()
- **功能**: 体积密度着色器评估。

### 降噪内核

#### filter_color_preprocess()
- **功能**: 降噪前颜色预处理，将去噪通道的值截断到 [0, 10000] 范围。

#### filter_guiding_preprocess()
- **功能**: 降噪引导数据预处理，从渲染缓冲区提取并缩放反照率、法线和光流数据。

#### filter_guiding_set_fake_albedo()
- **功能**: 设置伪反照率（0.5, 0.5, 0.5），用于无反照率通道时的降噪引导。

#### filter_color_postprocess()
- **功能**: 降噪后颜色后处理，将去噪结果缩放回原始采样计数下的值。

### 其他内核

#### integrator_shadow_catcher_count_possible_splits()
- **功能**: 统计可以进行阴影捕捉器路径分裂的状态数量。使用波前投票优化原子计数。

#### volume_guiding_filter_x() / volume_guiding_filter_y()
- **功能**: 体积散射概率引导的水平和垂直方向滤波。

## 依赖关系

- **内部头文件**:
  - `kernel/device/gpu/parallel_active_index.h` - 并行活跃索引构建
  - `kernel/device/gpu/parallel_prefix_sum.h` - 并行前缀和
  - `kernel/device/gpu/parallel_sorted_index.h` - 并行排序索引构建
  - `kernel/device/gpu/work_stealing.h` - 工作窃取与像素映射
  - `kernel/sample/lcg.h` - 线性同余随机数生成器
  - `kernel/tables.h` - 常量查找表
  - `kernel/integrator/state.h` - 积分器状态定义
  - `kernel/integrator/state_flow.h` - 积分器状态流转
  - `kernel/integrator/state_util.h` - 积分器状态工具
  - `kernel/integrator/init_from_bake.h` - 烘焙初始化
  - `kernel/integrator/init_from_camera.h` - 相机初始化
  - `kernel/integrator/intersect_*.h` - 各类光线求交
  - `kernel/integrator/shade_*.h` - 各类着色
  - `kernel/bake/bake.h` - 烘焙功能
  - `kernel/film/adaptive_sampling.h` - 自适应采样
  - `kernel/film/volume_guiding_denoise.h` - 体积引导降噪
  - `kernel/film/read.h` - 胶片读取
  - `kernel/device/metal/context_begin.h` / `context_end.h` - Metal 上下文（条件编译）
  - `kernel/device/oneapi/context_begin.h` / `context_end.h` - OneAPI 上下文（条件编译）
  - `kernel/device/hiprt/hiprt_kernels.h` - HIPRT 内核（条件编译）
- **被引用**:
  - `kernel/device/cuda/kernel.cu` - CUDA 后端
  - `kernel/device/hip/kernel.cpp` - HIP 后端
  - `kernel/device/hiprt/kernel.cpp` - HIPRT 后端
  - `kernel/device/metal/kernel.metal` - Metal 后端
  - `kernel/device/oneapi/kernel.cpp` - OneAPI 后端

## 实现细节

### 波前路径追踪架构

本文件体现了 Cycles 的波前路径追踪（wavefront path tracing）架构。与传统的逐像素 megakernel 不同，波前架构将路径追踪分解为多个独立的内核阶段：

1. **初始化阶段**: `integrator_init_from_camera` / `integrator_init_from_bake`
2. **求交阶段**: `integrator_intersect_*` 系列内核
3. **着色阶段**: `integrator_shade_*` 系列内核
4. **状态管理**: 排序、压缩、路径索引构建

每个内核通过 `path_index_array` 间接寻址路径状态，实现了路径的动态调度。

### 统一的内核模式

大多数内核遵循相同的调度模式：
```
const int global_index = ccl_gpu_global_id_x();
if (ccl_gpu_kernel_within_bounds(global_index, work_size)) {
    const int state = (path_index_array) ? path_index_array[global_index] : global_index;
    ccl_gpu_kernel_call(actual_function(nullptr, state, ...));
}
```
这一模式实现了：边界检查、可选的索引间接寻址、统一的函数调用约定。

### 跨平台兼容

文件通过 `ccl_gpu_kernel`、`ccl_gpu_kernel_signature`、`ccl_gpu_kernel_postfix` 等抽象宏实现跨 CUDA/HIP/Metal/OneAPI 平台的兼容。特定平台的差异通过条件编译处理（如 Metal 的 `__dummy_constant` 变通方案、OneAPI 的 `context_intersect_begin/end`、HIP 的 half float 写入问题）。

## 关联文件

| 文件 | 关系 |
|------|------|
| `kernel/device/gpu/parallel_active_index.h` | 提供并行路径索引压缩 |
| `kernel/device/gpu/parallel_prefix_sum.h` | 提供前缀和算法 |
| `kernel/device/gpu/parallel_sorted_index.h` | 提供排序索引构建 |
| `kernel/device/gpu/work_stealing.h` | 提供工作项到像素的映射 |
| `kernel/device/cuda/kernel.cu` | CUDA 后端入口，包含本文件 |
| `kernel/device/hip/kernel.cpp` | HIP 后端入口，包含本文件 |
| `kernel/device/metal/kernel.metal` | Metal 后端入口，包含本文件 |
| `kernel/device/oneapi/kernel.cpp` | OneAPI 后端入口，包含本文件 |
| `kernel/integrator/state.h` | 积分器状态结构定义 |
