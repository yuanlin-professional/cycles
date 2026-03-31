# pass.h / pass.cpp - 渲染通道定义与管理

## 概述

本文件定义了 Cycles 渲染器的渲染通道（Pass）系统。每个渲染通道代表一种输出数据类型（如组合通道、深度、法线、漫反射等），系统管理通道的类型信息、组件数、过滤和曝光属性、去噪支持和在缓冲区中的偏移计算。支持光照通道、数据通道、去噪通道、阴影捕捉通道和烘焙通道等多种类别。

## 类与结构体

### PassInfo
- **功能**: 渲染通道的属性信息描述
- **关键成员**:
  - `num_components` — 组件数（1/3/4）
  - `use_filter` — 是否应用采样过滤
  - `use_exposure` — 是否应用曝光调整
  - `is_written` — 是否由渲染管线写入（决定是否分配像素内存）
  - `scale` — 缩放因子
  - `divide_type` — 需要除以的关联通道类型（如运动向量除以权重）
  - `direct_type`, `indirect_type` — 直接/间接光照子通道
  - `use_compositing` — 是否需要合成才能读取
  - `use_denoising_albedo` — 是否使用去噪反照率
  - `support_denoise` — 是否支持去噪

### Pass
- **继承**: `Node`
- **功能**: 渲染通道节点，通过 NODE_SOCKET_API 暴露属性
- **关键成员**（通过 NODE_SOCKET_API）:
  - `type` — 通道类型（`PassType` 枚举）
  - `mode` — 通道模式（NOISY 或 DENOISED）
  - `name` — 通道名称
  - `include_albedo` — 是否包含反照率
  - `lightgroup` — 关联的灯光组
  - `is_auto_` — 是否为自动生成的通道
- **关键方法**:
  - `get_info()` — 获取当前通道的 PassInfo
  - `is_written()` — 检查通道是否被渲染管线写入
  - `get_info(type, mode, include_albedo, is_lightgroup)` — 静态方法，根据类型获取 PassInfo
  - `contains()` — 检查通道列表中是否包含指定类型
  - `find()` — 在通道列表中按名称或类型查找通道（两个重载）
  - `get_offset()` — 计算通道在渲染缓冲区中的像素偏移量
  - `get_type_enum()` — 获取类型枚举定义（静态，包含约 40 种通道类型）
  - `get_mode_enum()` — 获取模式枚举定义

### PassMode
- **功能**: 渲染通道模式枚举
- **值**: `NOISY`（原始噪声）, `DENOISED`（去噪后）

## 核心函数

- `pass_type_as_string()` — 将 PassType 枚举转换为字符串
- `pass_mode_as_string()` — 将 PassMode 枚举转换为字符串
- `is_volume_guiding_pass()` — 检查是否为体积引导通道（VOLUME_SCATTER 或 VOLUME_TRANSMIT）
- `operator<<` — Pass 和 PassMode 的流输出操作符

## 依赖关系

- **内部头文件**: `util/string.h`, `util/unique_ptr_vector.h`, `kernel/types.h`（PassType 枚举定义）, `graph/node.h`
- **cpp 额外引用**: `util/log.h`, `util/time.h`
- **被引用**: `scene/film.h`（Film 管理渲染通道列表）, `session/buffers.h`, `integrator/path_trace.cpp`, `integrator/path_trace_work.h`, `integrator/path_trace_tile.cpp`, `integrator/pass_accessor.h`

## 实现细节 / 关键算法

1. **通道类型注册**: `get_type_enum()` 使用静态 `NodeEnum` 注册约 40 种通道类型，分为光照通道（combined、emission、diffuse/glossy/transmission/volume 及其 direct/indirect 子通道）、数据通道（depth、normal、UV、motion 等）、去噪通道、阴影捕捉通道和烘焙通道。
2. **偏移量计算**: `get_offset()` 遍历所有通道，累加已写入通道的组件数作为偏移。未被写入的通道返回 `PASS_UNUSED`。
3. **合成通道**: 某些通道（如 diffuse、glossy、transmission）本身不直接写入缓冲区（`is_written = false`），而是通过组合其 direct/indirect 子通道在后处理阶段合成。
4. **反照率分离**: 光照通道支持 `include_albedo` 选项。关闭时设置 `divide_type` 为对应颜色通道（如 `PASS_DIFFUSE_COLOR`），后处理时除以反照率实现分离。
5. **体积引导通道**: `PASS_VOLUME_SCATTER` 和 `PASS_VOLUME_TRANSMIT` 在 NOISY 模式下使用 3 组件（高精度累加），DENOISED 模式下使用 1 组件（直接使用）。

## 关联文件

- `scene/film.h/.cpp` — 胶片/成像平面（管理渲染通道列表，调用 `update_passes()`）
- `scene/scene.h/.cpp` — 场景（持有 `passes` 列表）
- `kernel/types.h` — 内核类型（`PassType` 枚举定义）
- `session/buffers.h` — 渲染缓冲区（根据 Pass 信息分配内存）
- `integrator/pass_accessor.h` — 通道数据访问器
