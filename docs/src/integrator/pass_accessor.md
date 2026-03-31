# pass_accessor.h / pass_accessor.cpp - 渲染通道访问器基类

## 概述

本文件定义了 Cycles 渲染器的渲染通道数据访问抽象基类 `PassAccessor`。它提供统一的接口从渲染缓冲区读取各类渲染通道的像素数据（如深度、法线、合成、阴影捕捉等），并处理曝光、采样数缩放、通道组合等复杂逻辑。具体的 CPU 和 GPU 实现由子类提供。

## 类与结构体

### PassAccessor

- **继承**: 无（抽象基类）
- **功能**: 渲染通道数据的统一访问接口，负责根据通道类型分派到正确的内核处理函数，并管理曝光和采样数缩放。
- **关键成员**:
  - `pass_access_info_` — 通道访问信息
  - `exposure_` — 曝光值
  - `num_samples_` — 总采样数
- **关键方法**:
  - `get_render_tile_pixels()` — 从渲染缓冲区读取通道像素数据到目标位置（两个重载：自动/手动指定 `BufferParams`）
  - `set_render_tile_pixels()` — 将像素数据写入渲染缓冲区（用于烘焙读回）
  - `init_kernel_film_convert()` — 初始化 `KernelFilmConvert` 结构体，配置内核所需的全部参数

### PassAccessInfo

- **功能**: 描述要访问的渲染通道的元信息。
- **关键成员**:
  - `type` — 通道类型（`PassType`）
  - `mode` — 通道模式（`NOISY` 或 `DENOISED`）
  - `include_albedo` — 是否包含反照率
  - `is_lightgroup` — 是否为灯光组通道
  - `offset` — 在缓冲区中的偏移量
  - `use_approximate_shadow_catcher` — 是否使用近似阴影捕捉
  - `use_approximate_shadow_catcher_background` — 是否在背景上叠加近似阴影捕捉
  - `show_active_pixels` — 是否显示活跃像素

### Destination

- **功能**: 描述像素数据的输出目标。
- **关键成员**:
  - `pixels` / `pixels_half_rgba` — CPU 端浮点/半精度像素指针
  - `d_pixels` / `d_pixels_half_rgba` — 设备端像素指针
  - `num_components` — 每像素分量数
  - `offset` — 像素偏移
  - `pixel_offset` — 浮点偏移
  - `pixel_stride` — 像素步幅
  - `stride` — 行步幅

### Source

- **功能**: 描述像素数据的输入源（用于写入操作）。
- **关键成员**:
  - `pixels` — CPU 端浮点像素指针
  - `num_components` — 每像素分量数
  - `offset` — 像素偏移

## 核心函数

### get_render_tile_pixels()

根据通道类型和模式分派到对应的虚函数：

- **单通道（1 分量）**: `depth`、`mist`、`volume_majorant`、`sample_count`、`float`（通用）；体积引导通道使用 `rgbe`
- **Float3（3 分量）**: `light_path`、`shadow_catcher`、`rgbe`、`float3`（通用）
- **Float4（4 分量）**: `motion`、`cryptomatte`、`shadow_catcher_matte_with_shadow`、`combined`、`float4`（通用）

分派后调用 `pad_pixels()` 补齐分量（单通道复制到 RGB，缺失 alpha 填 1.0）。

### init_kernel_film_convert()

初始化 `KernelFilmConvert` 结构体的全部字段：
- 通道偏移与步幅
- 直接/间接/除法通道偏移（用于光路通道的组合计算）
- 特殊通道偏移（合成、采样计数、自适应采样、运动权重、阴影捕捉等）
- 缩放和曝光计算
- 自适应采样时的逐像素除法配置

### pad_pixels()

静态辅助函数，当目标分量数多于源分量数时填充额外通道：
- 1 分量 -> 3 分量：复制到 G、B
- 任意 -> 4 分量：alpha 设为 1.0

## 依赖关系

- **内部头文件**:
  - `scene/pass.h` — 通道类型和信息定义
  - `util/half.h` — 半精度浮点工具
  - `kernel/types.h` — 内核类型定义
  - `session/buffers.h` — 渲染缓冲区
- **被引用**:
  - `integrator/pass_accessor_cpu.h` — CPU 实现子类
  - `integrator/pass_accessor_gpu.h` — GPU 实现子类
  - `integrator/path_trace_work.h` — 路径追踪工作类
  - `integrator/path_trace.h` / `path_trace.cpp` — 路径追踪器

## 实现细节 / 关键算法

### 通道分派逻辑

`get_render_tile_pixels()` 中的分派遵循以下优先级：
1. 降噪后的通道直接使用通用 float/float3/float4 读取（已是最终值）
2. 特殊通道（depth、mist、motion、cryptomatte）有专用处理
3. 光路通道（有 `divide_type` 或 `direct_type`/`indirect_type`）使用 `light_path` 处理
4. 阴影捕捉相关通道有专用处理
5. 其余通道按分量数使用通用处理

### 采样数缩放

`init_kernel_film_convert()` 中的缩放逻辑：
- 当存在自适应采样计数通道时，由内核逐像素除以采样数
- 当不存在采样计数通道时，统一除以 `num_samples_`

## 关联文件

- `src/integrator/pass_accessor_cpu.h/.cpp` — CPU 侧实现
- `src/integrator/pass_accessor_gpu.h/.cpp` — GPU 侧实现
- `src/scene/pass.h` — `PassType`、`PassInfo`、`PassMode` 定义
- `src/kernel/types.h` — `KernelFilmConvert` 结构体定义
