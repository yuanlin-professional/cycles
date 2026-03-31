# output_driver.h - 离线渲染输出驱动接口

## 概述
本文件定义了 Cycles 渲染器的离线渲染输出驱动接口。`OutputDriver` 是一个抽象基类，宿主应用程序通过实现该接口来接收渲染完成的瓦片数据，可以将渲染缓冲区拷贝到应用程序内存或直接写入磁盘文件。虽然此接口也可用于交互式显示，但 `DisplayDriver` 针对该场景具有更高的效率。

## 类与结构体

### OutputDriver
- **功能**: 离线渲染缓冲区输出的抽象驱动接口。宿主应用实现此接口以接收渲染瓦片的最终或中间结果。

### OutputDriver::Tile
- **功能**: 表示一个渲染瓦片，包含瓦片的几何信息和像素数据访问接口。
- **关键成员**:
  - `offset` (`const int2`) — 瓦片在完整图像中的偏移位置
  - `size` (`const int2`) — 瓦片尺寸
  - `full_size` (`const int2`) — 完整图像的尺寸
  - `layer` (`const string`) — 渲染层名称
  - `view` (`const string`) — 视图名称
- **关键方法（纯虚函数）**:
  - `get_pass_pixels(pass_name, num_channels, pixels)` — 从渲染缓冲区获取指定通道的像素数据。调用者提供通道名称、通道数和目标缓冲区指针
  - `set_pass_pixels(pass_name, num_channels, pixels)` — 向渲染缓冲区设置指定通道的像素数据

### OutputDriver 的虚方法
- `write_render_tile(tile)` — **纯虚函数**。瓦片渲染完成后调用，宿主应用在此处理最终的瓦片数据
- `update_render_tile(tile)` — 渲染进行中的更新回调。返回 `true` 表示执行了更新操作。默认实现返回 `false`（不执行更新）
- `read_render_tile(tile)` — 用于烘焙（baking）流程，读取 `PASS_BAKE_PRIMITIVE`/`SEED`/`DIFFERENTIAL` 通道以确定每个像素的着色点。返回 `true` 表示读取了数据。默认实现返回 `false`

## 依赖关系
- **内部头文件**: `util/math.h`、`util/string.h`、`util/types.h`
- **外部库**: 无
- **被引用**: `session/session.cpp`、`integrator/path_trace_tile.h`、`hydra/output_driver.h`、`app/oiio_output_driver.h` 共 4 个文件

## 实现细节 / 关键算法
- **瓦片数据访问模式**: `Tile` 类的 `get_pass_pixels()` 和 `set_pass_pixels()` 提供了双向的像素数据访问。`get_pass_pixels()` 用于从渲染缓冲区读取数据（输出到文件或应用程序），`set_pass_pixels()` 用于向缓冲区写入数据（烘焙流程中提供输入数据）。
- **烘焙工作流**: `read_render_tile()` 在烘焙场景中被调用，宿主应用通过此方法提供烘焙所需的基元（primitive）、种子（seed）和微分（differential）信息，告知渲染器每个像素应对应哪个着色点。
- **与 DisplayDriver 的分工**: `OutputDriver` 面向离线渲染的最终输出，按瓦片粒度传递完整采样结果；`DisplayDriver` 面向交互式视口，提供实时渐进更新的低延迟显示。

## 关联文件
- `session/display_driver.h` — 交互式渲染的显示驱动接口（与本接口互补）
- `session/session.h` — `Session::set_output_driver()` 设置输出驱动
- `integrator/path_trace_tile.h` — `PathTraceTile` 继承自 `OutputDriver::Tile`，提供实际的像素数据访问实现
- `hydra/output_driver.h` — Hydra 渲染委托中的 `OutputDriver` 实现
- `app/oiio_output_driver.h` — 基于 OpenImageIO 的文件输出驱动实现
