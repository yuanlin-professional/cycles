# block_sizes.h - GPU 并行内核块大小常量定义

## 概述

本文件为 Cycles 渲染器的 GPU 并行算法定义了线程块大小(block size)常量。这些常量控制着并行活跃索引构建、并行前缀和以及并行排序索引等操作的线程组织方式。文件根据不同的 GPU 后端(HIP 与其他平台)设置不同的默认块大小，以适配硬件特性并优化性能。

## 核心宏定义

### GPU_PARALLEL_ACTIVE_INDEX_DEFAULT_BLOCK_SIZE
- **定义**: HIP 平台为 `1024`，其他平台为 `512`
- **功能**: 设定 `parallel_active_index` 算法中每个线程块的线程数量。HIP（AMD GPU）使用更大的块大小以充分利用其波前（wavefront）宽度。

### GPU_PARALLEL_PREFIX_SUM_DEFAULT_BLOCK_SIZE
- **定义**: HIP 平台为 `1024`，其他平台为 `512`
- **功能**: 设定 `parallel_prefix_sum` 算法中每个线程块的线程数量。

### GPU_PARALLEL_SORTED_INDEX_DEFAULT_BLOCK_SIZE
- **定义**: HIP 平台为 `1024`，其他平台为 `512`
- **功能**: 设定 `parallel_sorted_index` 算法中每个线程块的线程数量。

### GPU_PARALLEL_SORTED_INDEX_INACTIVE_KEY
- **定义**: `(~0)`（全1位模式，即 `0xFFFFFFFF`）
- **功能**: 用作排序索引算法中表示"非活跃状态"的哨兵键值。当路径状态不匹配目标内核时，使用此值标记为非活跃。

### GPU_PARALLEL_SORT_BLOCK_SIZE
- **定义**: `1024`（所有平台一致）
- **功能**: 设定基于本地原子操作的排序算法（bucket pass 和 write pass）中每个线程块的大小。

## 依赖关系

- **内部头文件**: 无（本文件为纯宏定义头文件，不依赖其他头文件）
- **被引用**:
  - `kernel/device/gpu/parallel_active_index.h` - 活跃索引并行构建
  - `kernel/device/gpu/parallel_sorted_index.h` - 排序索引并行构建
  - `integrator/path_trace_work_gpu.cpp` - GPU 路径追踪工作调度

## 实现细节

文件使用 `#ifdef __HIP__` 条件编译区分 AMD HIP 后端与其他后端（CUDA、Metal、OneAPI）。HIP 平台统一使用 1024 线程块大小，这是因为 AMD GPU 的波前大小为 64 个线程（相比 NVIDIA 的 32 个线程），更大的线程块有助于充分占满计算单元。其他平台默认使用 512 线程块大小，在占用率和寄存器压力之间取得平衡。

`GPU_PARALLEL_SORT_BLOCK_SIZE` 不受平台条件影响，始终为 1024，因为排序操作需要更大的块来保证足够的数据局部性。

## 关联文件

| 文件 | 关系 |
|------|------|
| `kernel/device/gpu/parallel_active_index.h` | 使用块大小常量进行活跃路径索引压缩 |
| `kernel/device/gpu/parallel_sorted_index.h` | 使用块大小常量和非活跃键值进行排序 |
| `kernel/device/gpu/parallel_prefix_sum.h` | 使用块大小常量进行前缀和计算 |
| `kernel/device/gpu/kernel.h` | 在内核启动时引用这些块大小常量 |
| `integrator/path_trace_work_gpu.cpp` | 主机端根据这些常量配置内核启动参数 |
