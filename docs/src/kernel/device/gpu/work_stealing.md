# work_stealing.h - GPU 工作窃取与像素映射

## 概述

本文件提供了 GPU 路径追踪中将全局工作索引映射到具体图块（tile）像素坐标和采样编号的工具函数。函数名中的"工作窃取"（work stealing）反映了 Cycles 渲染器的工作分配策略，即 GPU 线程从全局工作池中取出工作项进行处理。根据扰动距离（scrambling distance）的设置，函数会自动选择有利于性能的线程组织策略。

## 核心函数

### get_work_pixel()
- **签名**: `ccl_device_inline void get_work_pixel(const ccl_global KernelWorkTile *tile, const uint global_work_index, ccl_private uint *x, ccl_private uint *y, ccl_private uint *sample)`
- **功能**: 将线性的全局工作索引解码为二维像素坐标 (x, y) 和采样编号 (sample)。根据积分器的扰动距离参数选择不同的线程到像素映射策略。

**参数说明**:
- `tile`: 当前工作图块，包含起始坐标 (x, y)、尺寸 (w, h)、采样范围 (start_sample, num_samples)
- `global_work_index`: 在图块内的线性工作索引
- `x`, `y`: 输出的像素坐标（绝对坐标，已加上图块偏移）
- `sample`: 输出的采样编号（已加上起始采样偏移）

## 依赖关系

- **内部头文件**: 无显式 `#include`（依赖于编译环境中已有的 `KernelWorkTile` 和 `kernel_data` 定义）
- **被引用**:
  - `kernel/device/gpu/kernel.h` - 被 `integrator_init_from_camera` 和 `integrator_init_from_bake` 内核调用
  - `kernel/device/optix/kernel_osl.cu` - OptiX OSL 内核
  - `kernel/device/optix/kernel_osl_camera.cu` - OptiX OSL 相机内核

## 实现细节

### 两种线程映射策略

函数根据 `kernel_data.integrator.scrambling_distance` 的值选择不同的索引分解策略：

**策略一：同采样聚合（scrambling_distance < 0.9）**
```
sample_offset = global_work_index / tile_pixels
pixel_offset  = global_work_index % tile_pixels
```
将相同采样编号的线程组织在一起。当扰动距离较低时（如渐进式渲染的早期阶段），同一采样的像素之间有更好的数据局部性。

**策略二：同像素聚合（scrambling_distance >= 0.9）**
```
sample_offset = global_work_index % num_samples
pixel_offset  = global_work_index / num_samples
```
将相同像素的不同采样组织在一起。源码注释指出此策略在 CUDA 和 OptiX 上可提升约几个百分点的性能，原因是相同像素的多次采样共享几何数据和纹理缓存。

### 二维坐标还原

像素偏移量通过简单的除法和取余还原为二维坐标：
```
y_offset = pixel_offset / tile->w
x_offset = pixel_offset - y_offset * tile->w  // 等效于 pixel_offset % tile->w，避免除法
```

最终坐标加上图块的起始偏移得到绝对像素位置。

### 性能考量

避免使用取模运算符 `%`，改用乘法和减法（`pixel_offset - y_offset * tile->w`），因为整数除法和取模在 GPU 上代价较高。阈值 0.9 的选择基于实际性能测试结果。

## 关联文件

| 文件 | 关系 |
|------|------|
| `kernel/device/gpu/kernel.h` | 包含本文件，在初始化内核中调用 `get_work_pixel()` |
| `kernel/device/optix/kernel_osl.cu` | OptiX 后端使用本文件 |
| `kernel/device/optix/kernel_osl_camera.cu` | OptiX 相机内核使用本文件 |
| `kernel/integrator/init_from_camera.h` | 相机初始化逻辑，与本文件配合完成像素分配 |
| `kernel/integrator/init_from_bake.h` | 烘焙初始化逻辑，与本文件配合完成像素分配 |
