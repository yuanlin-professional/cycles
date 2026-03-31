# kernel_osl_camera.cu - OptiX OSL 相机初始化内核

## 概述

本文件实现了启用开放着色语言（OSL）支持的 OptiX 相机初始化内核。它定义了一个独立的 raygen 程序，用于从相机参数初始化光线路径。此内核作为渲染管线的第一个阶段，负责为每个像素采样生成初始光线并初始化积分器状态。本文件独立于 `kernel_osl.cu` 编译，以减少编译时间。

## 核心函数/宏定义

### 编译标志

- **`#define WITH_OSL`** - 启用 OSL 支持。

### raygen 程序

- **`__raygen__kernel_optix_integrator_init_from_camera()`** - 相机初始化光线生成程序。执行以下步骤：
  1. 从 `optixGetLaunchIndex().x` 获取全局线程索引
  2. 将 `kernel_params.path_index_array` 重解释为 `KernelWorkTile` 数组
  3. 根据全局索引计算图块索引和图块内工作索引
  4. 检查工作索引是否在有效范围内
  5. 通过 `get_work_pixel` 从工作图块中提取像素坐标 `(x, y)` 和采样编号 `sample`
  6. 调用 `integrator_init_from_camera` 初始化路径状态

## 依赖关系

- **内部头文件**:
  - `kernel/device/optix/compat.h` - OptiX 平台兼容层
  - `kernel/device/optix/globals.h` - 全局参数定义
  - `kernel/integrator/init_from_camera.h` - 相机初始化积分器实现
  - `kernel/device/gpu/work_stealing.h` - GPU 工作窃取/分配机制

- **被引用**: 本文件作为独立编译单元，不被其他源文件包含。

## 实现细节 / 关键算法

1. **工作图块分配**: 使用 `KernelWorkTile` 结构进行工作分配。全局索引通过除以 `max_tile_work_size` 得到图块索引，取模得到图块内的工作索引。每个图块包含一定数量的像素采样工作项。

2. **参数复用**: `kernel_params.path_index_array` 在此内核中被重解释为 `KernelWorkTile*` 指针。这是一种参数复用策略，避免在 `KernelParamsOptiX` 中增加额外字段。`kernel_params.num_tiles` 和 `kernel_params.max_tile_work_size` 字段专门为此内核使用。

3. **独立编译的原因**: 相机初始化内核不需要完整的着色和求交管线代码。将其独立编译可以：
   - 减少编译时间（不包含复杂的着色器头文件）
   - 减小单个编译单元的代码体积
   - 允许更灵活的 OptiX 管线模块组合

4. **边界检查**: `tile_work_index >= tile->work_size` 检查处理最后一个图块工作量不满的情况，多余的线程直接返回。

## 关联文件

- `kernel/device/optix/compat.h` - 兼容层
- `kernel/device/optix/globals.h` - 全局参数（提供 `kernel_params`）
- `kernel/device/optix/kernel_osl.cu` - OSL 完整内核（着色和求交阶段）
- `kernel/integrator/init_from_camera.h` - 相机初始化核心逻辑
- `kernel/device/gpu/work_stealing.h` - `get_work_pixel` 等工作分配函数
- `device/optix/device_impl.cpp` - 创建 OptiX 管线时加载本编译单元
