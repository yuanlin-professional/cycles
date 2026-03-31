# alloc.h - 着色器闭包内存分配器

## 概述

`alloc.h` 提供了 Cycles 渲染内核中着色器闭包（Closure）的内存分配基础设施。它负责在 `ShaderData` 的闭包数组中分配新的 `ShaderClosure` 槽位，以及为需要额外存储空间的闭包分配扩展内存。所有双向散射分布函数（BSDF）闭包的创建都依赖于此文件中的分配函数。

## 类与结构体

本文件不定义新的结构体，但操作以下核心类型：

- **`ShaderData`**: 着色器数据容器，包含闭包数组 `closure[]`、当前闭包数量 `num_closure` 以及剩余可用槽位 `num_closure_left`。
- **`ShaderClosure`**: 单个着色器闭包的数据结构，存储闭包类型 `type`、权重 `weight` 和采样权重 `sample_weight`。

## 核心函数

### closure_alloc()
- **签名**: `ccl_device ccl_private ShaderClosure *closure_alloc(ccl_private ShaderData *sd, const uint size, ClosureType type, Spectrum weight)`
- **功能**: 在着色器数据的闭包数组中分配一个新的闭包槽位。检查剩余可用槽位数，若为零则返回 `nullptr`。分配成功时设置闭包类型和权重，并更新计数器。参数 `size` 用于断言检查，确保不超过 `ShaderClosure` 的大小。

### closure_alloc_extra()
- **签名**: `ccl_device ccl_private void *closure_alloc_extra(ccl_private ShaderData *sd, const int size)`
- **功能**: 为需要额外参数空间的闭包分配扩展内存。从闭包数组的末尾向前分配，以 `sizeof(ShaderClosure)` 为单位进行对齐分配。这种设计保持了闭包数组的快速顺序迭代能力（相比链表方式更高效）。若剩余空间不足，会回滚上一次 `closure_alloc` 的分配并返回 `nullptr`。

### bsdf_alloc()
- **签名**: `ccl_device_inline ccl_private ShaderClosure *bsdf_alloc(ccl_private ShaderData *sd, const int size, Spectrum weight)`
- **功能**: 专用于双向散射分布函数（BSDF）闭包的分配。在调用 `closure_alloc` 之前进行权重验证：将负权重截断为零，计算采样权重（权重各通道的平均值的绝对值）。对于非体积着色器，当采样权重低于 `CLOSURE_WEIGHT_CUTOFF` 阈值时跳过分配以优化性能；对于体积着色器则跳过此截断（因为大体积低密度场景仍可能有显著贡献）。

## 依赖关系

- **内部头文件**:
  - `kernel/types.h` — 提供 `ShaderData`、`ShaderClosure`、`ClosureType`、`Spectrum` 等核心类型定义

- **被引用**:
  - `kernel/svm/closure.h` — SVM 着色器闭包创建
  - `kernel/osl/closures_setup.h` — OSL 着色器闭包创建
  - `kernel/integrator/subsurface.h` — 次表面散射积分器
  - `kernel/closure/bsdf_transparent.h` — 透明 BSDF 闭包
  - `kernel/closure/bsdf_ray_portal.h` — 光线传送门闭包
  - `kernel/closure/bssrdf.h` — 次表面散射闭包

## 实现细节 / 关键算法

### 闭包数组内存布局

闭包数组采用双端分配策略：
- **正常闭包**：从数组头部向后增长（`sd->closure[sd->num_closure]`）
- **额外数据**：从数组尾部向前增长（`sd->closure + sd->num_closure + sd->num_closure_left`）

这种布局使得两类数据共享同一数组空间，不需要额外的堆分配，适合 GPU 内核的执行环境。

### 权重截断策略

`bsdf_alloc` 中使用 `CLOSURE_WEIGHT_CUTOFF` 进行权重截断，但体积着色器（`SD_IS_VOLUME_SHADER_EVAL` 标志）除外。此外，通过 `isfinite_safe` 的间接检查（使用 `>=` 比较），可以自动过滤非有限浮点值（NaN、Inf）。

## 关联文件

- `kernel/types.h` — 核心类型定义
- `kernel/svm/closure.h` — SVM 闭包系统，频繁调用本文件的分配函数
- `kernel/osl/closures_setup.h` — OSL 闭包系统，频繁调用本文件的分配函数
