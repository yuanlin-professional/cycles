# parallel_prefix_sum.h - GPU 并行前缀和计算

## 概述

本文件实现了 GPU 上的前缀和（prefix sum / exclusive scan）算法。该算法将计数器数组转换为偏移量数组，是排序索引构建的前置步骤。当前实现为单线程串行版本，因为其处理的数据量（场景中着色器的数量）通常较小，并非性能瓶颈。

## 核心函数

### gpu_parallel_prefix_sum()
- **签名**: `__device__ void gpu_parallel_prefix_sum(const int global_id, ccl_global int *counter, ccl_global int *prefix_sum, const int num_values)`
- **功能**: 对 `counter[]` 数组执行独占前缀和（exclusive prefix sum），结果写入 `prefix_sum[]`。同时将 `counter[]` 数组清零，为下一轮计数做准备。仅由 `global_id == 0` 的线程执行实际计算，其余线程直接返回。

**参数说明**:
- `global_id`: 全局线程索引，仅索引为 0 的线程执行计算
- `counter`: 输入计数器数组，每个元素表示对应键值的出现次数，计算完成后被清零
- `prefix_sum`: 输出前缀和数组，`prefix_sum[i]` 为 `counter[0..i-1]` 的累加和
- `num_values`: 数组长度（通常等于场景中着色器的数量）

## 依赖关系

- **内部头文件**:
  - `util/atomic.h` - 原子操作工具（本文件包含但未直接使用，保持接口一致性）
- **被引用**:
  - `kernel/device/gpu/kernel.h` - 通过 `prefix_sum` 内核包装层调用

## 实现细节

### 算法逻辑

```
offset = 0
for i in 0..num_values:
    new_offset = offset + counter[i]
    prefix_sum[i] = offset
    counter[i] = 0
    offset = new_offset
```

这是一个标准的独占前缀和：`prefix_sum[i]` 存储 `counter[0]` 到 `counter[i-1]` 的总和（不包含 `counter[i]` 本身）。执行完毕后 `counter[]` 被清零，因为在波前路径追踪的排序流程中，计数器会被后续的原子加法操作重新累积。

### 为何是串行实现

源码注释 "TODO: actually make this work in parallel" 表明这是一个已知的简化。当前设计选择串行的原因：
- 数组大小等于场景着色器数量（`max_shaders`），通常为几十到几百个元素
- 对于如此小的数据集，并行化的启动开销可能超过计算本身的收益
- 仅由一个线程执行，其余线程通过 `global_id != 0` 检查立即返回

### 在排序流水线中的角色

前缀和是路径排序的中间步骤：
1. **计数阶段**: 原子操作统计每个着色器键值的路径数量 -> `counter[]`
2. **前缀和阶段**: 本函数将计数转为起始偏移 -> `prefix_sum[]`（同时清零 `counter[]`）
3. **写入阶段**: `gpu_parallel_sorted_index_array` 利用 `prefix_sum[]` 确定每条路径的写入位置

## 关联文件

| 文件 | 关系 |
|------|------|
| `kernel/device/gpu/kernel.h` | 包含本文件并定义 `prefix_sum` GPU 内核 |
| `kernel/device/gpu/parallel_sorted_index.h` | 排序索引构建，依赖前缀和结果 |
| `kernel/device/gpu/block_sizes.h` | 定义 `GPU_PARALLEL_PREFIX_SUM_DEFAULT_BLOCK_SIZE` |
| `util/atomic.h` | 原子操作工具库 |
