# parallel_active_index.h - GPU 并行活跃路径索引压缩

## 概述

本文件实现了 GPU 上的并行流压缩（stream compaction）算法：给定一个路径状态数组，并行构建出所有"活跃"状态的索引数组。该算法是 Cycles 波前路径追踪架构的关键基础设施，用于在每个内核调度周期中快速筛选出需要处理的路径。实现涵盖 CUDA/HIP、Metal 和 OneAPI 三种后端的差异化处理。

## 核心函数

### gpu_parallel_active_index_array_impl() [OneAPI 版本]
- **签名**: `template<typename IsActiveOp> void gpu_parallel_active_index_array_impl(const uint num_states, ccl_global int *ccl_restrict indices, ccl_global int *ccl_restrict num_indices, IsActiveOp is_active_op)`
- **功能**: OneAPI/SYCL 后端的并行活跃索引构建实现。支持两种模式：`WITH_ONEAPI_SYCL_HOST_TASK` 下退化为串行循环；否则利用 SYCL sub_group 的 `exclusive_scan_over_group` 执行 warp 级前缀和。

### gpu_parallel_active_index_array_impl() [CUDA/HIP 版本]
- **签名**: `template<typename IsActiveOp> __device__ void gpu_parallel_active_index_array_impl(const uint num_states, ccl_global int *indices, ccl_global int *num_indices, IsActiveOp is_active_op)`
- **功能**: CUDA/HIP 后端的并行活跃索引构建实现。使用 `ccl_gpu_ballot` 进行 warp 级投票和 `popcount` 计算前缀偏移。

### gpu_parallel_active_index_array_impl() [Metal 版本]
- **签名**: `void gpu_parallel_active_index_array_impl(const uint num_states, ccl_global int *indices, ccl_global int *num_indices, const uint is_active, const uint blocksize, const int thread_index, const uint state_index, const int ccl_gpu_warp_size, const int thread_warp, const int warp_index, const int num_warps, threadgroup int *warp_offset)`
- **功能**: Metal 后端版本。由于 Metal 不支持 C++ 模板 lambda，`is_active` 判定结果和硬件参数由调用方预先计算并显式传入。

### gpu_parallel_active_index_array (宏)
- **签名**: `gpu_parallel_active_index_array(num_states, indices, num_indices, is_active_op)`
- **功能**: 跨平台统一调用接口。Metal 版本负责提取 `simd_lane_index`、`simd_group_index` 等硬件参数后调用实现函数；CUDA/HIP 和 OneAPI 版本直接转发调用。

## 依赖关系

- **内部头文件**:
  - `kernel/device/gpu/block_sizes.h` - 线程块大小常量
  - `util/atomic.h` - 原子操作（`atomic_fetch_and_add_uint32`）
- **被引用**:
  - `kernel/device/gpu/kernel.h` - 被所有路径索引构建内核调用

## 实现细节

### 三阶段并行流压缩算法

算法分为三个同步阶段：

**阶段一：Warp 级前缀和**
- 每个线程判断其对应的路径状态是否活跃
- 在 warp 内部通过投票指令（`ccl_gpu_ballot`）获得活跃掩码
- 使用 `popcount` 计算当前线程之前有多少个活跃线程（`thread_offset`）
- 每个 warp 的最后一个线程将本 warp 的活跃数存入共享内存 `warp_offset[]`

**阶段二：Block 级前缀和**（`ccl_gpu_syncthreads` 同步后）
- 线程块中最后一个线程串行扫描 `warp_offset[]` 数组，将每个 warp 的活跃计数转换为偏移量
- 同时通过全局原子加法获取本线程块在全局输出数组中的起始偏移
- 全局偏移存入 `warp_offset[num_warps]`

**阶段三：写入**（再次同步后）
- 每个活跃线程计算其全局写入位置：`block_offset + warp_offset[warp_index] + thread_offset`
- 将其路径状态索引写入 `indices[]` 数组

### 共享内存需求

每个线程块需要 `sizeof(int) * (num_warps + 1)` 字节的共享内存。其中 `num_warps` 个槽位存储各 warp 的偏移量，额外 1 个槽位存储全局块偏移。

### 平台差异

| 特性 | CUDA/HIP | Metal | OneAPI |
|------|----------|-------|--------|
| Warp 投票 | `ccl_gpu_ballot` | 通过宏预计算 `is_active` | `exclusive_scan_over_group` |
| 共享内存 | `extern ccl_gpu_shared` | `threadgroup int *` | `group_local_memory` |
| 同步屏障 | `ccl_gpu_syncthreads` | `ccl_gpu_syncthreads` | `ccl_gpu_local_syncthreads`（更快的本地屏障） |
| 模板支持 | 支持 | 不支持（参数显式传入） | 支持 |

OneAPI 特别注明使用本地屏障（`ccl_gpu_local_syncthreads`）而非全局屏障，因为此处仅需同步共享内存写入，本地屏障性能更优。

## 关联文件

| 文件 | 关系 |
|------|------|
| `kernel/device/gpu/block_sizes.h` | 提供 `GPU_PARALLEL_ACTIVE_INDEX_DEFAULT_BLOCK_SIZE` |
| `kernel/device/gpu/kernel.h` | 调用本算法构建各类路径索引数组 |
| `kernel/device/gpu/parallel_sorted_index.h` | 排序版本的流压缩，解决类似问题 |
| `util/atomic.h` | 提供跨平台原子操作 |
