# aov.h - 任意输出变量(AOV)节点的SVM实现

## 概述

`aov.h` 实现了着色器虚拟机(SVM)中的任意输出变量(AOV, Arbitrary Output Variable)节点。AOV 节点允许用户在着色器中输出自定义数据到渲染通道，用于合成阶段的后处理。该文件提供了颜色 AOV 和数值 AOV 两种输出类型，以及用于判断是否应写入 AOV 数据的检查函数。

## 核心函数

### `svm_node_aov_check`
- **签名**: `ccl_device_inline bool svm_node_aov_check(const uint32_t path_flag, const ccl_global float *render_buffer)`
- **功能**: 检查当前路径是否应该写入 AOV 数据。
- **条件**: 仅当渲染缓冲区有效 (`render_buffer != nullptr`) 且当前路径为主要路径（透明背景路径且未完成单次通道写入）时返回 `true`。这确保 AOV 数据仅在首次可见的着色点写入一次。

### `svm_node_aov_color`
- **签名**: `template<uint node_feature_mask, typename ConstIntegratorGenericState> void svm_node_aov_color(KernelGlobals kg, ConstIntegratorGenericState state, float *stack, const uint4 node, float *render_buffer)`
- **功能**: 将 float3 颜色值写入 AOV 渲染通道。
- **流程**: 从栈中加载 float3 值 (`node.y`)，调用 `film_write_aov_pass_color` 写入渲染缓冲区中指定通道 (`node.z`)。

### `svm_node_aov_value`
- **签名**: `template<uint node_feature_mask, typename ConstIntegratorGenericState> void svm_node_aov_value(KernelGlobals kg, ConstIntegratorGenericState state, float *stack, const uint4 node, float *render_buffer)`
- **功能**: 将单个浮点值写入 AOV 渲染通道。
- **流程**: 从栈中加载 float 值 (`node.y`)，调用 `film_write_aov_pass_value` 写入渲染缓冲区中指定通道 (`node.z`)。

## 依赖关系

- **内部头文件**:
  - `kernel/film/aov_passes.h` — AOV 渲染通道写入函数 (`film_write_aov_pass_color`, `film_write_aov_pass_value`)
  - `kernel/svm/util.h` — SVM 栈操作工具
- **被引用**: `kernel/svm/svm.h`

## 实现细节 / 关键算法

1. **主路径限定**: AOV 数据仅在 `PATH_RAY_TRANSPARENT_BACKGROUND` 路径上写入，且 `PATH_RAY_SINGLE_PASS_DONE` 未被设置。这意味着 AOV 仅记录相机直接可见（或透过透明物体可见）的第一个非透明着色点的数据，避免在反射/折射等间接路径上重复写入。

2. **特征掩码守卫**: 使用 `IF_KERNEL_NODES_FEATURE(AOV)` 宏包裹实际写入逻辑，确保仅在内核启用 AOV 特征时执行。

3. **渲染缓冲区安全检查**: 在调用 AOV 写入函数前，先通过 `svm_node_aov_check` 验证渲染缓冲区指针有效性，防止空指针访问。

## 关联文件

- `kernel/film/aov_passes.h` — AOV 通道的底层写入实现
- `kernel/svm/svm.h` — SVM 主调度器
