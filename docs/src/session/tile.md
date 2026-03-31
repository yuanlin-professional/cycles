# tile.h / tile.cpp - 瓦片管理与磁盘分块读写

## 概述
本文件实现了 Cycles 渲染器的瓦片（Tile）管理系统，负责将完整的渲染图像分割为较小的瓦片进行分块渲染，并管理瓦片数据的磁盘读写。`TileManager` 是核心类，它处理瓦片的调度序列、尺寸计算、缓冲区参数与 EXR 图像元数据之间的双向序列化，以及通过 OpenImageIO 将瓦片渲染结果写入磁盘临时文件和从磁盘读回完整帧数据。此机制支持大图渲染中的内存优化和中间结果持久化。

## 类与结构体

### Tile
- **功能**: 描述单个渲染瓦片的几何信息。
- **关键成员**:
  - `x` / `y` (`int`) — 瓦片在完整图像中的位置（包含过扫描区域）
  - `width` / `height` (`int`) — 瓦片的实际尺寸（包含过扫描）
  - `window_x` / `window_y` (`int`) — 有效渲染窗口在瓦片内的偏移
  - `window_width` / `window_height` (`int`) — 有效渲染窗口的尺寸

### TileManager
- **功能**: 瓦片调度与磁盘 I/O 管理器。不可拷贝、不可移动。
- **公开成员**:
  - `full_buffer_written_cb` (`std::function<void(string_view)>`) — 瓦片文件关闭后的回调，允许宿主应用跟踪磁盘文件
- **公开方法**:
  - `reset_scheduling(params, tile_size)` — 重置调度状态，根据图像尺寸和瓦片大小计算网格布局
  - `update(params, scene)` — 更新缓冲区参数和场景配置，配置 EXR 图像规格，处理自适应采样的过扫描
  - `set_temp_dir(temp_dir)` — 设置临时文件目录
  - `get_num_tiles()` — 获取总瓦片数
  - `has_multiple_tiles()` — 是否有多个瓦片
  - `get_tile_overscan()` — 获取过扫描像素数
  - `next()` — 前进到下一个瓦片，返回是否还有瓦片
  - `done()` — 是否所有瓦片已处理完毕
  - `get_current_tile()` — 获取当前瓦片信息
  - `get_size()` — 获取完整图像尺寸
  - `write_tile(tile_buffers)` — 将瓦片渲染缓冲区写入磁盘 EXR 文件
  - `finish_write_tiles()` — 完成瓦片写入（补写空瓦片、关闭文件、触发回调）
  - `has_written_tiles()` — 是否有瓦片已写入磁盘
  - `read_full_buffer_from_disk(filename, buffers, denoise_params)` — 从磁盘瓦片文件读取完整帧数据
  - `compute_render_tile_size(suggested)` — 计算兼容图像保存的有效瓦片大小
- **常量**:
  - `IMAGE_TILE_SIZE = 128` — EXR 文件中的图像瓦片大小
  - `MAX_TILE_SIZE = 8192` — 最大支持的瓦片尺寸（受 GPU 纹理分配限制）
- **保护成员**:
  - `tile_file_unique_part_` (`string`) — 瓦片文件名唯一标识（进程ID + 对象地址 + 实例计数器）
  - `tile_size_` (`int2`) — 当前瓦片尺寸
  - `overscan_` (`int`) — 过扫描像素数（自适应采样时为 4，否则为 0）
  - `buffer_params_` (`BufferParams`) — 缓冲区参数副本
  - `tile_state_` — 瓦片调度状态（网格布局、当前索引、当前瓦片）
  - `write_state_` — 瓦片文件写入状态（文件索引、文件名、图像规格、输出句柄、已写瓦片数）

## 核心函数

### 图像规格与缓冲区参数的序列化

#### `exr_channel_names_for_passes()`
- **功能**: 为渲染通道生成严格有序的 EXR 通道名。使用 8 位数字前缀（如 `00000000`）确保字母序与缓冲区内存偏移一致，允许直接将渲染缓冲区数据转储到磁盘再读回。

#### `node_socket_to_image_spec_atttributes()` / `node_socket_from_image_spec_atttributes()`
- **功能**: Node 套接字属性与 OIIO ImageSpec 元数据之间的双向转换。支持 ENUM、STRING、INT、FLOAT、BOOLEAN 类型。

#### `buffer_params_to_image_spec_atttributes()` / `buffer_params_from_image_spec_atttributes()`
- **功能**: `BufferParams`（包括所有 `BufferPass`）与 ImageSpec 元数据之间的完整序列化/反序列化。通道列表手动展开序列化，因为不在标准的 Node 套接字覆盖范围内。

#### `configure_image_spec_from_buffer()`
- **功能**: 从缓冲区参数创建完整的 ImageSpec，包括通道名、元数据和可选的瓦片 IO 配置。

### 瓦片调度

#### `get_tile_for_index()`
- **功能**: 根据瓦片索引计算瓦片几何信息。使用行优先遍历，考虑过扫描区域，裁剪到图像边界。

#### `compute_render_tile_size()`
- **功能**: 将建议的瓦片大小对齐到 `IMAGE_TILE_SIZE` 的倍数（以满足 OpenEXR 的对齐要求），并限制在 `MAX_TILE_SIZE` 以内。

### 瓦片磁盘 I/O

#### `open_tile_output()`
- **功能**: 创建临时 EXR 文件用于瓦片写入。文件名格式为 `cycles-tile-buffer-<unique>-<index>.exr`。

#### `write_tile()`
- **功能**: 将瓦片渲染缓冲区写入已打开的 EXR 文件。处理过扫描时将有效窗口区域像素拷贝到连续内存块（解决 OIIO bug）。使用 `write_tiles()` 进行分块写入。

#### `finish_write_tiles()`
- **功能**: 完成瓦片文件写入。EXR 要求所有瓦片都存在，因此补写未渲染的空瓦片。关闭文件后触发 `full_buffer_written_cb` 回调并递增文件索引。

#### `read_full_buffer_from_disk()`
- **功能**: 从瓦片文件读取完整帧数据。反序列化 `BufferParams` 和 `DenoiseParams`，重置渲染缓冲区并读取所有像素数据。

## 枚举与常量
- `ATTR_PASSES_COUNT` — 元数据属性名：通道数量
- `ATTR_PASS_SOCKET_PREFIX_FORMAT` — 元数据属性名前缀格式：单个通道
- `ATTR_BUFFER_SOCKET_PREFIX` — 元数据属性名前缀：缓冲区参数
- `ATTR_DENOISE_SOCKET_PREFIX` — 元数据属性名前缀：降噪参数
- `g_instance_index` — 全局原子计数器，用于生成唯一瓦片文件名

## 依赖关系
- **内部头文件**: `session/buffers.h`、`session/session.h`、`graph/node.h`、`scene/background.h`、`scene/bake.h`、`scene/film.h`、`scene/integrator.h`、`scene/scene.h`、`util/image.h`、`util/string.h`、`util/unique_ptr.h`、`util/log.h`、`util/path.h`、`util/system.h`、`util/time.h`、`util/types.h`
- **外部库**: OpenImageIO（通过 `util/image.h`），`<atomic>`、`<functional>`
- **被引用**: `session/session.h`、`session/tile.cpp`、`integrator/render_scheduler.cpp`、`integrator/path_trace.cpp` 共 4 个文件

## 实现细节 / 关键算法

### 瓦片遍历顺序
当前使用简单的行优先（row-major）遍历顺序。代码中有 TODO 注释提到可以考虑使用希尔伯特螺旋（Hilbert spiral）等空间填充曲线优化遍历顺序，但认为在大瓦片场景下收益有限。

### 过扫描（Overscan）机制
当启用自适应采样且非烘焙模式时，每个瓦片向外扩展 4 像素的过扫描区域。过扫描区域参与渲染但不写入最终输出，目的是避免自适应采样在瓦片边界产生的伪影。瓦片几何中 `window_*` 字段表示有效区域，`x/y/width/height` 包含过扫描的完整区域。

### EXR 瓦片对齐
渲染瓦片大小必须是 `IMAGE_TILE_SIZE`（128）的倍数，因为 EXR 文件的瓦片必须对齐。`compute_render_tile_size()` 使用 `align_up()` 确保对齐，同时不超过 `MAX_TILE_SIZE`（8192）以避免 GPU 纹理分配失败。

### 唯一文件名生成
使用 `进程ID + 对象内存地址 + 全局原子计数器` 的组合生成唯一标识，解决以下问题：
- 多个 Cycles 进程并行运行
- 同一进程中多次创建 TileManager（对象可能重用相同地址）
- 原子计数器溢出但仍有活跃实例

### OIIO Bug 规避
`write_tile()` 中对有过扫描的瓦片，将有效窗口区域的像素拷贝到连续内存块再写入。这是对 OIIO 中 tiled write 接口的一个已知 bug（oiio#3176）的规避措施。

## 关联文件
- `session/session.h` — `Session` 持有 `TileManager` 并驱动瓦片渲染流程
- `session/buffers.h` — `BufferParams` 和 `RenderBuffers`，瓦片管理器序列化/反序列化的核心数据
- `integrator/render_scheduler.h` — `RenderScheduler` 与瓦片管理器协作调度渲染工作
- `integrator/path_trace.h` — `PathTrace` 使用瓦片参数配置单瓦片渲染
- `scene/integrator.h` — 提供降噪参数和自适应采样配置
