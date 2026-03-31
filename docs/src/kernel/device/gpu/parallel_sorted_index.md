# parallel_sorted_index.h - GPU 并行排序索引构建

## 概述

本文件实现了 GPU 上按键值排序的活跃路径索引构建算法。与 `parallel_active_index.h` 的无序压缩不同，本文件不仅筛选活跃路径，还按照着色器排序键将路径分组排列。这使得相同着色器的路径在内存中连续存放，提升着色内核执行时的波前一致性和缓存命中率。文件提供了两种实现：基于全局原子操作的通用版本和基于本地原子操作的优化排序版本。

## 核心函数

### gpu_parallel_sort_bucket_pass()
- **签名**: `ccl_device_inline void gpu_parallel_sort_bucket_pass(const uint num_states, const uint partition_size, const uint max_shaders, const uint queued_kernel, ccl_global ushort *d_queued_kernel, ccl_global uint *d_shader_sort_key, ccl_global int *partition_key_offsets, ccl_gpu_shared int *buckets, const ushort local_id, const ushort local_size, const uint grid_id)`
- **功能**: 两趟排序的第一趟（桶计数趟）。将状态数组分为多个分区（partition），每个线程块负责一个分区，统计分区内各着色器键值的出现次数。结果存入 `partition_key_offsets`，用于第二趟确定写入位置。
- **条件编译**: 仅在定义 `__KERNEL_LOCAL_ATOMIC_SORT__` 时可用。

### gpu_parallel_sort_write_pass()
- **签名**: `ccl_device_inline void gpu_parallel_sort_write_pass(const uint num_states, const uint partition_size, const uint max_shaders, const uint queued_kernel, const int num_states_limit, ccl_global int *indices, ccl_global ushort *d_queued_kernel, ccl_global uint *d_shader_sort_key, ccl_global int *partition_key_offsets, ccl_gpu_shared int *local_offset, const ushort local_id, const ushort local_size, const uint grid_id)`
- **功能**: 两趟排序的第二趟（写入趟）。根据第一趟计算的分区偏移量，将每条活跃路径的索引写入排序后的全局 `indices[]` 数组中。使用本地原子操作跟踪每个着色器桶的写入进度。

### gpu_parallel_sorted_index_array()
- **签名**: `template<typename GetKeyOp> __device__ void gpu_parallel_sorted_index_array(const uint state_index, const uint num_states, const int num_states_limit, ccl_global int *indices, ccl_global int *num_indices, ccl_global int *key_counter, ccl_global int *key_prefix_sum, GetKeyOp get_key_op)`
- **功能**: 基于全局原子操作的排序索引构建。每个活跃线程通过 `get_key_op` 获取其排序键值，然后对 `key_prefix_sum[key]` 执行原子加法获得写入位置。当写入位置超过 `num_states_limit` 时，递增 `key_counter[key]` 以便后续迭代处理溢出部分。

## 依赖关系

- **内部头文件**:
  - `kernel/device/gpu/block_sizes.h` - 提供 `GPU_PARALLEL_SORTED_INDEX_INACTIVE_KEY` 和块大小常量
  - `util/atomic.h` - 原子操作（`atomic_fetch_and_add_uint32`、`atomic_fetch_and_add_uint32_shared`、`atomic_store_local`、`atomic_load_local`）
- **被引用**:
  - `kernel/device/gpu/kernel.h` - 被 `integrator_sorted_paths_array`、`integrator_sort_bucket_pass`、`integrator_sort_write_pass` 内核调用

## 实现细节

### 全局原子排序方案 (gpu_parallel_sorted_index_array)

这是较简单的实现，适用于所有平台：

1. 每个线程通过 `get_key_op(state_index)` 获取排序键值（通常是着色器索引）
2. 非活跃状态返回 `GPU_PARALLEL_SORTED_INDEX_INACTIVE_KEY`（`~0`），直接跳过
3. 活跃线程对 `key_prefix_sum[key]` 执行原子加法，获得唯一的写入索引
4. 如果索引未超过 `num_states_limit`，将 `state_index` 写入 `indices[index]`
5. 如果超过限制，递增 `key_counter[key]`，将此状态推迟到下一轮迭代处理

该方案的缺点是大量全局原子操作可能造成竞争，源码注释 "TODO: there may be ways to optimize this to avoid this many atomic ops" 点明了这一问题。

### 本地原子排序方案 (bucket_pass + write_pass)

这是优化后的两趟排序实现（`__KERNEL_LOCAL_ATOMIC_SORT__`），通过分区和本地共享内存减少全局原子竞争：

**第一趟 (bucket_pass)**:
1. 将状态数组分割为固定大小的分区（`partition_size`）
2. 在共享内存中分配 `buckets[]` 数组（大小为 `max_shaders`）并初始化为 0
3. 遍历分区内的状态，匹配目标内核的路径对其着色器键值桶进行本地原子计数
4. 最后一个线程计算分区内各键值的局部前缀和并存入 `partition_key_offsets`
5. 额外存储分区的活跃状态总数

**第二趟 (write_pass)**:
1. 计算每个分区各键值的全局偏移：累加所有前驱分区的活跃状态总数
2. 将全局偏移与分区局部偏移合并，存入共享内存 `local_offset[]`
3. 再次遍历分区内的状态，匹配的路径通过本地原子加法获取写入位置
4. 将路径索引写入全局 `indices[]` 数组

### 溢出处理

`num_states_limit` 参数限制了单次迭代可处理的最大状态数量。超出限制的状态不会被丢弃，而是通过递增 `key_counter` 确保在后续迭代中被重新处理。这一机制防止了内存越界，同时保证所有路径最终都能被处理。

## 关联文件

| 文件 | 关系 |
|------|------|
| `kernel/device/gpu/block_sizes.h` | 提供 `GPU_PARALLEL_SORTED_INDEX_DEFAULT_BLOCK_SIZE` 和 `GPU_PARALLEL_SORTED_INDEX_INACTIVE_KEY` |
| `kernel/device/gpu/parallel_prefix_sum.h` | 计算前缀和，作为本算法的前置步骤 |
| `kernel/device/gpu/parallel_active_index.h` | 无序版本的流压缩，解决类似但更简单的问题 |
| `kernel/device/gpu/kernel.h` | 包含本文件并定义排序相关的 GPU 内核 |
| `util/atomic.h` | 提供全局和本地原子操作 |
