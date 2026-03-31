# output_driver.h / output_driver.cpp - Hydra渲染代理的输出驱动实现

## 概述

本文件实现了 Cycles 渲染器在 Hydra渲染代理框架中的输出驱动层。`HdCyclesOutputDriver` 继承自 Cycles 的 `OutputDriver`，负责将渲染结果的各个 AOV（Arbitrary Output Variable）通道数据写入到 Hydra 的渲染缓冲区中。它处理除显示 AOV 外的所有渲染通道数据传输，支持直接映射和格式转换两种写入路径。

## 类与结构体

### HdCyclesOutputDriver

- **继承**: `CCL_NS::OutputDriver`
- **功能**: 将 Cycles 渲染瓦片（Tile）中的各 AOV 通道数据传输到对应的 Hydra 渲染缓冲区。
- **关键成员**:
  - `_renderParam` (`HdCyclesSession*`) — 渲染会话参数，用于获取 AOV 绑定列表和显示绑定
- **关键方法**:
  - `write_render_tile()` — 最终渲染完成时调用，写入瓦片数据并将所有渲染缓冲区标记为已收敛
  - `update_render_tile()` — 渲染过程中的增量更新，将瓦片数据写入渲染缓冲区

## 核心函数

### `update_render_tile()`
- **功能**: 遍历所有 AOV 绑定，将瓦片数据写入对应的渲染缓冲区。实现两种写入路径：
  1. **直接映射路径**: 当瓦片偏移为零、尺寸匹配渲染缓冲区且格式为 Float32 系列时，直接映射缓冲区内存并写入
  2. **通用路径**: 先读取像素到临时缓冲区，再通过 `WritePixels()` 写入渲染缓冲区，支持子区域更新和 ID 通道识别
- **跳过逻辑**: 跳过显示 AOV 绑定（已由 `HdCyclesDisplayDriver` 处理）和无效格式绑定

### `write_render_tile()`
- **功能**: 调用 `update_render_tile()` 完成数据写入后，遍历所有渲染缓冲区并调用 `SetConverged(true)` 标记渲染收敛。

## 依赖关系

- **内部头文件**:
  - `hydra/config.h` — 命名空间配置
  - `session/output_driver.h` — Cycles `OutputDriver` 基类
  - `hydra/render_buffer.h` — `HdCyclesRenderBuffer`
  - `hydra/session.h` — `HdCyclesSession`
- **外部头文件**: 无额外外部依赖
- **被引用**:
  - `hydra/output_driver.cpp`, `hydra/render_pass.cpp`

## 实现细节 / 关键算法

1. **显示 AOV 跳过**: 当某个 AOV 绑定是显示绑定（`_renderParam->GetDisplayAovBinding()`）且其渲染缓冲区正在使用 GPU 资源（`IsResourceUsed()`）时，跳过该绑定，因为显示通道已由 `HdCyclesDisplayDriver` 通过 PBO 路径更新。

2. **直接映射优化**: 当瓦片完全覆盖渲染缓冲区（偏移为零且尺寸匹配）并且格式为 Float32 时，直接映射缓冲区内存调用 `tile.get_pass_pixels()`，避免了额外的像素数据拷贝。

3. **ID 通道识别**: 对于 `primId`、`elementId`、`instanceId` 等 ID 类型的 AOV，通过 `isId` 标志传递给 `WritePixels()`，以确保整数 ID 值在写入时不会被错误地进行浮点转换。

4. **缺失通道处理**: 当 `get_pass_pixels()` 未找到请求的通道时，对 `elementId` 静默忽略（因为这是标准 AOV 但 Cycles 未实现），对其他通道输出运行时错误。

## 关联文件

- `src/session/output_driver.h` — Cycles `OutputDriver` 基类定义
- `src/hydra/display_driver.h` — 显示 AOV 的 GPU 路径驱动
- `src/hydra/render_buffer.h` — `HdCyclesRenderBuffer`（目标写入缓冲区）
- `src/hydra/render_pass.cpp` — 创建 `HdCyclesOutputDriver` 实例
- `src/hydra/session.h` — `HdCyclesSession`（AOV 绑定管理）
