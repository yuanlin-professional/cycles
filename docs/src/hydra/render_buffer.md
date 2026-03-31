# render_buffer.h / render_buffer.cpp - Hydra渲染缓冲区（AOV输出目标）

## 概述

本文件实现了 Cycles 的 Hydra 渲染缓冲区，作为 AOV（Arbitrary Output Variable）的输出目标。`HdCyclesRenderBuffer` 管理像素数据的分配、映射、格式转换和收敛状态，支持多种像素格式（UNorm8、SNorm8、Float16、Float32、Int32）之间的写入转换。

渲染缓冲区是渲染通道与输出驱动之间的数据中转层，承接 Cycles 渲染结果并供 Hydra 客户端读取。

## 类与结构体

### HdCyclesRenderBuffer

- **继承**: `PXR_NS::HdRenderBuffer`
- **功能**: 管理单个 AOV 通道的像素数据存储、内存映射、格式转换和收敛状态
- **关键成员**:
  - `_width`, `_height` (`unsigned int`) -- 缓冲区宽高
  - `_format` (`HdFormat`) -- 像素格式
  - `_dataSize` (`size_t`) -- 数据总字节数
  - `_data` (`vector<uint8_t>`) -- 像素数据存储
  - `_resource` (`VtValue`) -- GPU 资源句柄（用于显示驱动路径）
  - `_resourceUsed` (`atomic_bool`) -- 标记 GPU 资源是否被读取过
  - `_mapped` (`atomic_int`) -- 映射计数器（支持嵌套映射）
  - `_converged` (`atomic_bool`) -- 渲染收敛标志
- **关键方法**:
  - `Allocate()` -- 分配缓冲区，仅支持 2D（depth 必须为 1），相同大小时跳过重分配
  - `Map()` / `Unmap()` -- 映射/解映射像素数据供读取，设置了 GPU 资源时返回 `nullptr`
  - `WritePixels()` -- 从 float 源数据写入目标格式像素，支持偏移和通道数适配
  - `SetConverged()` / `IsConverged()` -- 设置和查询收敛状态
  - `GetResource()` / `SetResource()` -- GPU 纹理资源的存取
  - `Finalize()` -- 从 AOV 绑定中移除自身，防止输出驱动继续写入

## 核心函数

### 匿名命名空间内的格式转换器

- `SimpleConversion` -- float 到 float 的恒等转换
- `IdConversion` -- float 到 int32 的 ID 转换（值减 1，适配 Hydra 的 primId 约定）
- `UInt8Conversion` -- float 到 uint8（乘 255）
- `SInt8Conversion` -- float 到 int8（乘 127）
- `HalfConversion` -- float 到 half（调用 `float_to_half_image`）
- `writePixels<SrcT, DstT, Convertor>()` -- 模板函数，按行列遍历将源像素转换写入目标缓冲区，自动处理尺寸裁剪和通道数差异

## 依赖关系

- **内部头文件**: `hydra/render_buffer.h`, `hydra/session.h`, `util/half.h`
- **外部依赖**: `pxr/imaging/hd/renderBuffer.h`, `pxr/base/gf/vec3i.h`, `pxr/base/gf/vec4f.h`
- **被引用**: `render_pass.cpp`, `render_delegate.cpp`, `output_driver.cpp`, `display_driver.cpp`

## 实现细节 / 关键算法

1. **延迟分配**: `Allocate()` 仅记录尺寸和格式，实际 `_data` 容器在首次 `Map()` 时才 `resize` 到目标大小，减少不必要的内存分配。

2. **WritePixels 格式分派**: `WritePixels()` 根据 `_format` 枚举值进行 switch 分支，选择对应的转换器模板实例。对于 `HdFormatInt32` 的 ID AOV 特殊处理：使用 `IdConversion` 将渲染器输出的 1-based ID 转换为 0-based。

3. **线程安全**: `_mapped` 和 `_converged` 均使用 `std::atomic`，支持渲染线程写入与客户端线程读取的并发访问。

4. **资源模式**: 当通过 `SetResource()` 设置了 GPU 纹理资源时，`Map()` 返回 `nullptr`，像素数据通过 GPU 路径传递而非 CPU 内存映射。

## 关联文件

- `hydra/output_driver.h` -- 通过 `WritePixels()` 向缓冲区写入渲染结果
- `hydra/display_driver.h` -- 通过 `SetResource()` 设置 GPU 纹理资源
- `hydra/render_pass.h` -- 管理缓冲区的收敛状态
- `hydra/session.h` -- AOV 绑定管理
