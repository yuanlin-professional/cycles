# session - 渲染会话管理

## 概述

`session/` 模块是 Cycles 渲染引擎的顶层管理层，负责连接场景（Scene）、设备（Device）和积分器（Integrator），管理渲染的完整生命周期。该模块提供以下核心功能：

- **渲染会话**：`Session` 类作为 Cycles 的主入口点，管理渲染线程、设备创建、场景更新和渲染流程控制
- **渲染缓冲区**：定义渲染缓冲区的参数、通道布局和设备端存储
- **瓦片管理**：将全帧图像分割为瓦片，支持渐进式磁盘写入和跨计算机分布式渲染
- **驱动接口**：定义 `DisplayDriver` 和 `OutputDriver` 抽象接口，供宿主应用（如 Blender）实现
- **离线降噪**：支持独立的基于文件的降噪管线
- **图像合并**：支持合并多个 OpenEXR 多层渲染结果

## 目录结构

| 文件名 | 类型 | 说明 |
|--------|------|------|
| `CMakeLists.txt` | 构建 | CMake 构建配置 |
| `session.h/cpp` | 核心 | 渲染会话主类：管理渲染线程、设备、场景更新和渲染循环 |
| `buffers.h/cpp` | 核心 | 渲染缓冲区：`BufferParams`（参数与通道布局）、`RenderBuffers`（设备端像素存储） |
| `tile.h/cpp` | 核心 | 瓦片管理器：`Tile`（瓦片描述）、`TileManager`（瓦片调度与磁盘 I/O） |
| `display_driver.h/cpp` | 接口 | 显示驱动抽象接口，宿主应用实现此接口以在 viewport 中显示渲染结果 |
| `output_driver.h` | 接口 | 输出驱动抽象接口，宿主应用实现此接口以接收离线渲染的瓦片数据 |
| `denoising.h/cpp` | 工具 | 独立降噪管线：从磁盘读取图像、执行降噪、写回结果 |
| `merge.h/cpp` | 工具 | 图像合并器：合并多个 OpenEXR 多层渲染文件 |

## 核心类与数据结构

### Session

渲染会话主类，是 Cycles 渲染引擎的统一入口。核心职责：

- **设备管理**：根据 `SessionParams` 创建渲染设备和降噪设备
- **场景管理**：持有 `Scene` 实例，在渲染前同步场景更新到设备
- **渲染线程**：在独立线程中运行渲染循环，支持暂停/恢复/取消
- **渲染编排**：通过 `RenderScheduler` 获取渲染工作，交由 `PathTrace` 执行
- **进度报告**：通过 `Progress` 对象向外部报告渲染进度

关键成员：
- `device` / `denoise_device_`：渲染设备和降噪设备（均为 `unique_ptr<Device>`）
- `scene`：场景实例
- `params`：会话参数（`SessionParams`）
- `progress`：进度跟踪器
- `tile_manager_`：瓦片管理器
- `render_scheduler_`：渲染调度器
- `path_trace_`：路径追踪器

渲染线程状态机：
```
SESSION_THREAD_WAIT  ──(start)──>  SESSION_THREAD_RENDER  ──(完成/取消)──>  SESSION_THREAD_END
       ^                                   |
       └──────(暂停/等待新工作)─────────────┘
```

主渲染循环（`run_main_render_loop`）：
```
while (!done):
  1. delayed_reset_buffer_params()   // 处理延迟重置
  2. update_scene()                  // 同步场景到设备
  3. tile_manager_.next()            // 推进到下一个瓦片
  4. render_scheduler_.get_render_work()  // 获取渲染工作
  5. run_wait_for_work()             // 若无工作或已暂停则等待
  6. path_trace_->render(work)       // 执行渲染
```

### SessionParams

会话参数，控制渲染行为的顶层配置：

| 参数 | 类型 | 说明 |
|------|------|------|
| `device` | `DeviceInfo` | 渲染设备信息 |
| `denoise_device` | `DeviceInfo` | 降噪设备信息 |
| `headless` | `bool` | 是否无界面渲染 |
| `background` | `bool` | 是否后台（离线）渲染 |
| `samples` | `int` | 目标采样数（默认 1024） |
| `pixel_size` | `int` | 像素大小（用于 HiDPI 显示） |
| `threads` | `int` | CPU 线程数（0 = 自动） |
| `time_limit` | `double` | 渲染时间限制（秒，0 = 无限制） |
| `use_auto_tile` | `bool` | 是否启用自动瓦片 |
| `tile_size` | `int` | 瓦片大小（默认 2048） |
| `use_resolution_divider` | `bool` | 是否启用分辨率缩放 |
| `shadingsystem` | `ShadingSystem` | 着色系统（SVM 或 OSL） |
| `temp_dir` | `string` | 临时文件目录（用于瓦片磁盘写入） |
| `use_sample_subset` | `bool` | 是否渲染采样子集（分布式渲染） |
| `sample_subset_offset` | `int` | 采样子集起始偏移 |
| `sample_subset_length` | `int` | 采样子集长度 |

### BufferParams

渲染缓冲区参数，描述渲染缓冲区的尺寸、通道布局和在全帧中的位置：

- **物理缓冲区尺寸**：`width` / `height` — 实际分配的缓冲区大小
- **可见窗口**：`window_x/y/width/height` — 缓冲区中可见的区域（窗口外为 overscan）
- **全帧位置**：`full_x/y/width/height` — 在完整渲染图像中的偏移和尺寸（用于边框渲染）
- **通道配置**：`passes` — `BufferPass` 列表，定义所有渲染通道及其在缓冲区中的偏移
- **运行时字段**：`offset`、`stride`、`pass_stride` — 在 `update_passes()` 后计算得出

`BufferPass` 描述单个渲染通道：类型（`PassType`）、模式（噪声/降噪）、名称、偏移等。

### RenderBuffers

设备端渲染缓冲区，持有实际的像素数据：

- `params`：缓冲区参数
- `buffer`：`device_vector<float>` — 设备端浮点像素缓冲区
- `reset()` / `zero()`：重置或清零缓冲区
- `copy_from_device()` / `copy_to_device()`：在主机和设备间复制数据

辅助函数 `render_buffers_host_copy_denoised()` 用于在渲染缓冲区之间复制降噪通道数据。

### TileManager

瓦片管理器，将全帧图像分割为瓦片并管理渲染进度和磁盘 I/O：

- **瓦片调度**：`reset_scheduling()` 初始化瓦片网格，`next()` / `done()` 遍历所有瓦片
- **磁盘写入**：`write_tile()` 将瓦片渲染缓冲区写入 OpenEXR 文件，`finish_write_tiles()` 关闭文件
- **全帧读取**：`read_full_buffer_from_disk()` 从磁盘读取所有瓦片并合并为全帧缓冲区
- **Overscan**：在瓦片边界外渲染额外像素，用于降噪时避免接缝

关键常量：
- `IMAGE_TILE_SIZE = 128`：图像文件中的瓦片大小
- `MAX_TILE_SIZE = 8192`：最大支持的瓦片大小（受 GPU 纹理分配限制）

### Tile

瓦片描述符，定义瓦片在图像中的位置和尺寸：

- `x/y/width/height`：瓦片位置和大小（包含 overscan）
- `window_x/y/width/height`：瓦片内的可见窗口（不含 overscan）

### DisplayDriver

显示驱动抽象接口，宿主应用（如 Blender）实现此接口以在 viewport 中实时显示渲染结果。支持三种纹理更新方式：

1. **纹理缓冲区映射**：`map_texture_buffer()` / `unmap_texture_buffer()` — CPU 端写入映射的 GPU 纹理
2. **图形互操作**：`GraphicsInteropBuffer` — GPU 直接写入，避免 CPU-GPU 数据传输
3. **像素复制**：通过 `PathTraceDisplay::copy_pixels_to_texture()` 逐片复制

`GraphicsInteropDevice` 描述显示设备信息（OpenGL/Vulkan/Metal 及 UUID），用于判断是否可与渲染设备进行互操作。

`GraphicsInteropBuffer` 封装原生图形 API 的像素缓冲区句柄，支持 OpenGL PBO、Vulkan VkBuffer 和 Metal MTLBuffer。

关键方法：
- `update_begin()` / `update_end()`：标记纹理更新周期
- `draw()`：使用原生图形 API 绘制渲染结果
- `zero()`：清空显示缓冲区
- `next_tile_begin()`：通知下一个瓦片即将开始

### OutputDriver

输出驱动抽象接口，宿主应用实现此接口以接收离线渲染的瓦片数据：

- `write_render_tile()`：瓦片渲染完成后写出结果
- `update_render_tile()`：渲染过程中的增量更新（可选）
- `read_render_tile()`：为烘焙读取输入数据（可选）

内部类 `OutputDriver::Tile` 提供按通道名称访问像素数据的接口（`get_pass_pixels` / `set_pass_pixels`）。

### DenoiserPipeline

独立的基于文件的降噪管线，用于对已渲染的图像序列执行降噪：

- 输入一组帧文件路径，输出降噪后的文件
- 内部创建独立的设备和降噪器实例
- `DenoiseTask` 处理单帧的加载、降噪和保存
- `DenoiseImage` 解析 OpenEXR 通道结构，管理像素数据

### ImageMerger

图像合并工具，将多个 OpenEXR 多层渲染文件合并为一个文件。用于分布式渲染后合并各节点的渲染结果。

## 模块架构

### 渲染生命周期

```
1. 创建阶段
   Session(params, scene_params)
     ├─ 创建 Device（渲染设备）
     ├─ 创建 Device（降噪设备，可选）
     ├─ 创建 Scene
     ├─ 创建 TileManager
     ├─ 创建 RenderScheduler
     └─ 创建 PathTrace

2. 配置阶段
   session->set_output_driver(driver)     // 设置输出驱动
   session->set_display_driver(driver)    // 设置显示驱动
   session->set_samples(N)               // 设置采样数
   session->reset(params, buffer_params) // 重置并配置缓冲区

3. 渲染阶段
   session->start()                      // 启动渲染线程
     └─ thread_render()
         └─ run_main_render_loop()
              ├─ update_scene()          // 场景同步
              ├─ get_render_work()       // 调度
              └─ path_trace_->render()   // 执行

4. 交互阶段（viewport）
   session->draw()                       // 绘制当前结果
   session->reset(...)                   // 参数变更时重置
   session->set_pause(true/false)        // 暂停/恢复

5. 结束阶段
   session->cancel()                     // 取消渲染
   session->wait()                       // 等待线程结束
   ~Session()                            // 清理资源
```

### 组件关系图

```
                    ┌──────────────────────┐
                    │     宿主应用          │
                    │   (Blender 等)       │
                    └──────┬───────────────┘
                           │ 实现接口
              ┌────────────┼────────────┐
              v            v            v
      DisplayDriver   OutputDriver   Session API
              │            │            │
              └────────────┼────────────┘
                           │
                    ┌──────v──────┐
                    │   Session   │
                    └──────┬──────┘
                           │ 持有
          ┌────────────────┼────────────────┐
          v                v                v
       Device          Scene         TileManager
          │                                 │
          │            ┌────────────────────┘
          v            v
    ┌─────────────────────────┐
    │    RenderScheduler      │ ◄── 决定渲染工作
    └────────────┬────────────┘
                 v
    ┌─────────────────────────┐
    │      PathTrace          │ ◄── 执行渲染管线
    ├─────────────────────────┤
    │ PathTraceWork[] (每设备) │
    │ Denoiser                │
    │ PathTraceDisplay        │
    │ OutputDriver            │
    └─────────────────────────┘
```

## 依赖关系

### 上游依赖（本模块依赖）

| 模块 | 用途 |
|------|------|
| `device/` | 设备抽象层：`Device`、`DeviceInfo`、`device_memory`、`DeviceQueue` |
| `scene/` | 场景描述：`Scene`、`SceneParams`、`Pass`、`Shader` |
| `integrator/` | 渲染执行：`PathTrace`、`RenderScheduler`、`Denoiser`、`PassAccessor` |
| `graph/` | 节点系统：`Node` 基类（用于 `BufferParams` 序列化） |
| `kernel/` | 内核类型定义：`PassType`、`PASS_NUM` 等 |
| `util/` | 工具库：线程、进度、统计、图像 I/O、字符串等 |
| OpenImageIO | 外部库：EXR 文件读写（瓦片 I/O 和降噪管线） |

### 下游依赖（依赖本模块）

| 模块 | 用途 |
|------|------|
| `blender/` | Blender 集成：创建 `Session`，实现 `DisplayDriver` 和 `OutputDriver` |
| `app/` | 命令行应用：`cycles_standalone` 使用 `Session` 执行离线渲染 |
| `integrator/` | 积分器使用 `BufferParams`、`RenderBuffers`、`TileManager`、`DisplayDriver`、`OutputDriver` |

## 参见

- `src/integrator/` — 积分器模块，执行实际的路径追踪、降噪和显示更新
- `src/device/` — 设备抽象层，提供 CPU/GPU 计算设备
- `src/scene/` — 场景描述，定义渲染通道和胶片参数
- `src/kernel/` — 内核实现，定义渲染通道类型和缓冲区布局常量
