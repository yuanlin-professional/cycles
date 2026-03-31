# buffers.h / buffers.cpp - 渲染缓冲区管理与渲染通道数据布局

## 概述
本文件定义了 Cycles 渲染器中渲染缓冲区（Render Buffer）的核心数据结构与操作。它负责管理渲染过程中像素数据的存储布局，包括渲染通道（Pass）的偏移计算、缓冲区参数配置以及设备（Device）端与主机端之间的数据传输。在 session 模块中，这些类是渲染结果数据在内存中的基本载体，被积分器（Integrator）、降噪器（Denoiser）、路径追踪等多个子系统广泛引用。

## 类与结构体

### BufferPass
- **继承**: `Node`（使用 Node API 实现序列化/反序列化，但并非真正的场景节点）
- **功能**: 描述渲染缓冲区中的单个渲染通道（Pass），记录通道类型、模式、名称及在缓冲区中的偏移量。
- **关键成员**:
  - `type` (`PassType`) — 渲染通道类型（如 `PASS_COMBINED`、`PASS_DEPTH` 等）
  - `mode` (`PassMode`) — 通道模式，`NOISY`（含噪声）或 `DENOISED`（已降噪）
  - `name` (`ustring`) — 通道名称
  - `include_albedo` (`bool`) — 是否包含反照率数据
  - `lightgroup` (`ustring`) — 光照组名称
  - `offset` (`int`) — 通道在缓冲区中的浮点数偏移量，`PASS_UNUSED` 表示未使用
- **关键方法**:
  - `BufferPass(const Pass *scene_pass)` — 从场景通道对象构造缓冲区通道
  - `get_info()` — 返回通道信息（`PassInfo`），包括组件数等
  - `operator==` / `operator!=` — 比较两个缓冲区通道是否相同

### BufferParams
- **继承**: `Node`（使用 Node API 实现序列化/反序列化）
- **功能**: 描述渲染缓冲区的完整参数，包括物理尺寸、窗口区域（可见区域）、完整图像中的位置、所有渲染通道的布局，以及曝光等渲染属性。
- **关键成员**:
  - `width` / `height` (`int`) — 物理缓冲区的宽高
  - `window_x` / `window_y` / `window_width` / `window_height` (`int`) — 可见窗口区域，窗口外的部分视为过扫描（overscan）
  - `full_x` / `full_y` / `full_width` / `full_height` (`int`) — 在完整渲染图像中的偏移与尺寸（用于边界渲染）
  - `offset` / `stride` (`int`) — 运行时字段，经 `update_passes()` 或 `update_offset_stride()` 后有效
  - `pass_stride` (`int`) — 每像素的浮点数步长（所有通道组件总数）
  - `passes` (`vector<BufferPass>`) — 缓冲区中所有通道的列表
  - `layer` / `view` (`ustring`) — 渲染层和视图名称
  - `samples` (`int`) — 采样数
  - `exposure` (`float`) — 曝光值
  - `use_approximate_shadow_catcher` / `use_transparent_background` (`bool`) — 阴影捕捉器与透明背景配置
  - `pass_offset_[]` (`int[kNumPassOffsets]`) — 按通道类型和模式索引的偏移量快速查找表
- **关键方法**:
  - `update_passes()` — 根据已有通道列表重新计算偏移和步长
  - `update_passes(const unique_ptr_vector<Pass> &scene_passes)` — 从场景通道列表创建缓冲区通道并更新参数
  - `get_pass_offset(PassType, PassMode)` — 获取指定类型和模式通道在缓冲区中的偏移量
  - `find_pass(string_view name)` / `find_pass(PassType, PassMode)` — 按名称或类型查找通道
  - `get_actual_display_pass(...)` — 获取实际显示通道，会在存在阴影捕捉器遮罩通道时替换合成通道
  - `update_offset_stride()` — 根据完整图像坐标更新 offset 和 stride
  - `modified(const BufferParams &other)` — 判断参数是否发生变化

### RenderBuffers
- **功能**: 封装实际的渲染缓冲区数据，管理设备端与主机端之间的浮点数组传输。
- **关键成员**:
  - `params` (`BufferParams`) — 当前缓冲区参数
  - `buffer` (`device_vector<float>`) — 设备端浮点数缓冲区
- **关键方法**:
  - `RenderBuffers(Device *device)` — 构造函数，绑定到指定设备
  - `reset(const BufferParams &params)` — 根据新参数重新分配缓冲区
  - `zero()` — 将缓冲区清零（设备端）
  - `copy_from_device()` — 从设备拷贝数据到主机
  - `copy_to_device()` — 从主机拷贝数据到设备

## 核心函数

### `render_buffers_host_copy_denoised()`
- **功能**: 在主机端将已降噪的渲染通道从源缓冲区拷贝到目标缓冲区。
- **参数**: 目标/源 `RenderBuffers` 及其对应的 `BufferParams`，可选的源像素偏移量。
- **实现细节**: 构建通道偏移映射表，逐像素拷贝所有 `DENOISED` 模式的通道数据（当前仅支持 RGBA 四通道），支持不同通道集的缓冲区之间的拷贝。

### 内部辅助函数
- `pass_type_mode_to_index()` — 将通道类型和模式转换为 `pass_offset_[]` 数组索引（类型 * 2 + 降噪偏移）
- `pass_to_index()` — 从 `BufferPass` 获取索引

## 依赖关系
- **内部头文件**: `device/memory.h`、`graph/node.h`、`scene/pass.h`、`kernel/types.h`、`util/string.h`、`util/unique_ptr.h`、`util/vector.h`、`device/device.h`、`util/log.h`
- **外部库**: 无
- **被引用**: `session/session.h`、`session/session.cpp`、`session/tile.h`、`session/denoising.h`、`integrator/path_trace.h`、`integrator/path_trace_work.h`、`integrator/path_trace_work_cpu.cpp`、`integrator/path_trace_work_gpu.cpp`、`integrator/denoiser.cpp`、`integrator/denoiser_gpu.cpp`、`integrator/pass_accessor.cpp`、`integrator/render_scheduler.h`、`scene/bake.cpp`、`device/cpu/device_impl.cpp`、`app/cycles_standalone.cpp` 等共 24 个文件

## 实现细节 / 关键算法
- **通道偏移快速查找**: `pass_offset_[]` 数组大小为 `PASS_NUM * 2`，通过 `pass_type * 2 + (mode == DENOISED ? 1 : 0)` 索引，实现 O(1) 时间的通道偏移查找。当存在多个相同类型和模式的通道时，存储最低偏移值。
- **阴影捕捉器显示替换**: `get_actual_display_pass()` 在请求 `PASS_COMBINED` 且存在 `PASS_SHADOW_CATCHER_MATTE` 通道时，自动返回遮罩通道用于显示。
- **缓冲区内存布局**: 缓冲区按 `width * pass_stride` 行 x `height` 列组织，每像素存储所有通道的浮点分量。

## 关联文件
- `scene/pass.h` — 场景级渲染通道定义（`Pass`、`PassType`、`PassMode`、`PassInfo`）
- `session/tile.h` — 瓦片管理器，使用 `BufferParams` 配置分块渲染
- `session/session.h` — 渲染会话，持有 `BufferParams` 并协调缓冲区生命周期
- `integrator/path_trace.h` — 路径追踪器，使用 `RenderBuffers` 进行渲染
- `device/memory.h` — `device_vector` 模板，管理设备端内存
