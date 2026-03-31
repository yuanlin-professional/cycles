# integrator - 积分器（渲染调度）

## 概述

`integrator/` 模块是 Cycles 渲染引擎的核心调度层，负责管理完整的路径追踪渲染流程。该模块协调以下关键职责：

- **路径追踪执行**：在单个或多个设备（CPU/GPU）上调度和执行路径追踪采样
- **渲染调度**：根据设备性能和用户设置智能安排渲染工作（采样数、分辨率缩放、显示更新频率等）
- **自适应采样**：根据像素收敛状态动态终止已收敛区域的采样，提升渲染效率
- **降噪**：集成 OpenImageDenoise (OIDN) 和 OptiX 降噪器，支持 CPU 和 GPU 降噪
- **工作负载均衡**：在多设备渲染时动态分配工作量
- **通道访问**：从渲染缓冲区中提取和转换各类渲染通道数据
- **着色器求值**：用于位移贴图和背景光等场景预处理的着色器离线求值
- **路径引导**：可选的基于 OpenPGL 的路径引导功能，用于优化采样策略

## 目录结构

| 文件名 | 类型 | 说明 |
|--------|------|------|
| `CMakeLists.txt` | 构建 | CMake 构建配置 |
| `adaptive_sampling.h/cpp` | 核心 | 自适应采样逻辑：采样对齐、收敛检测触发时机 |
| `denoiser.h/cpp` | 核心 | 降噪器抽象基类，定义降噪接口和工厂方法 |
| `denoiser_gpu.h/cpp` | 实现 | GPU 降噪器基类，处理设备端缓冲区管理和通用 GPU 降噪流程 |
| `denoiser_oidn.h/cpp` | 实现 | 基于 OpenImageDenoise 的 CPU 降噪器实现 |
| `denoiser_oidn_gpu.h/cpp` | 实现 | 基于 OpenImageDenoise 的 GPU 降噪器实现 |
| `denoiser_optix.h/cpp` | 实现 | 基于 NVIDIA OptiX 的 GPU 降噪器实现 |
| `guiding.h` | 数据 | 路径引导参数定义（`GuidingParams`），配置 OpenPGL 引导策略 |
| `pass_accessor.h/cpp` | 核心 | 渲染通道访问器抽象基类，从渲染缓冲区读取/写入各类通道数据 |
| `pass_accessor_cpu.h/cpp` | 实现 | CPU 端通道访问器，使用 CPU 内核函数进行胶片转换 |
| `pass_accessor_gpu.h/cpp` | 实现 | GPU 端通道访问器，通过设备队列执行胶片转换内核 |
| `path_trace.h/cpp` | 核心 | 路径追踪主控类，编排完整的渲染管线 |
| `path_trace_display.h/cpp` | 核心 | 渲染显示包装器，管理 `DisplayDriver` 的线程安全和状态跟踪 |
| `path_trace_tile.h/cpp` | 适配 | `OutputDriver::Tile` 的实现，桥接路径追踪器与输出驱动 |
| `path_trace_work.h/cpp` | 核心 | 单设备路径追踪工作抽象基类，定义采样、显示更新、自适应采样等接口 |
| `path_trace_work_cpu.h/cpp` | 实现 | CPU 路径追踪工作实现，逐像素调度，使用 TBB 线程并行 |
| `path_trace_work_gpu.h/cpp` | 实现 | GPU 路径追踪工作实现，按工作瓦片调度，管理波前积分器状态 |
| `render_scheduler.h/cpp` | 核心 | 渲染调度器，决定每次迭代的工作内容（采样数、降噪时机、显示更新等） |
| `shader_eval.h/cpp` | 工具 | 着色器离线求值，用于位移贴图、背景光、曲线阴影透明度等预计算 |
| `tile.h/cpp` | 工具 | 设备端瓦片大小计算，根据图像尺寸和路径状态数优化瓦片尺寸 |
| `work_balancer.h/cpp` | 工具 | 多设备工作负载均衡器，根据设备时间和占用率分配工作权重 |
| `work_tile_scheduler.h/cpp` | 工具 | 工作瓦片调度器，将大瓦片分割为适合设备执行的小工作瓦片 |

## 核心类与数据结构

### PathTrace

路径追踪主控类，是整个渲染管线的编排者。负责：

- 管理一个或多个 `PathTraceWork` 实例（每个渲染设备对应一个）
- 按顺序执行渲染管线的各个步骤：初始化缓冲区 -> 路径追踪 -> 自适应采样 -> 降噪 -> 显示更新 -> 工作再平衡 -> 写入瓦片
- 持有 `Denoiser`、`PathTraceDisplay`、`OutputDriver` 等子系统
- 管理多设备之间的渲染缓冲区复制与合并
- 支持路径引导（OpenPGL）的训练数据收集与引导场更新

关键成员：
- `device_` / `denoise_device_`：渲染设备和降噪设备（可能是 `MultiDevice`）
- `path_trace_works_`：每个子设备对应的 `PathTraceWork` 列表
- `work_balance_infos_`：多设备工作负载均衡信息
- `render_scheduler_`：渲染调度器引用
- `denoiser_`：降噪器实例

### RenderScheduler

渲染调度器，决定每次渲染迭代应执行的工作内容。核心职责：

- 计算每次迭代的采样数（基于时间估算和设备占用率）
- 管理分辨率缩放器（viewport 导航时降低分辨率以保证交互性）
- 决定何时执行降噪、何时更新显示、何时进行工作再平衡
- 处理自适应采样的渐进噪声阈值（progressive noise floor）
- 跟踪渲染状态（已渲染样本数、时间限制、收敛状态等）
- 生成渲染工作描述 `RenderWork`

内部类 `TimeWithAverage` 跟踪各任务（路径追踪、降噪、显示更新等）的壁钟时间和平均时间，用于启发式调度决策。

### RenderWork

渲染工作描述符，封装单次渲染迭代中需要执行的所有工作：

- `path_trace`：路径追踪采样参数（起始样本、采样数、偏移）
- `adaptive_sampling`：自适应采样滤波和阈值
- `tile`：瓦片写入和瓦片级降噪
- `display`：显示更新和降噪结果使用
- `rebalance`：是否需要多设备再平衡
- `resolution_divider`：分辨率缩放因子

### PathTraceWork

单设备路径追踪工作的抽象基类。每个渲染设备拥有一个实例，负责：

- 在其设备上执行路径追踪采样（`render_samples`）
- 将渲染结果复制到显示纹理（`copy_to_display`）
- 执行自适应采样收敛检测和滤波
- 管理该设备的渲染缓冲区切片（大瓦片的一个子区域）

具有两个具体实现：
- **PathTraceWorkCPU**：CPU 实现，逐像素调度，使用线程级 `kernel_globals` 副本实现 TBB 并行
- **PathTraceWorkGPU**：GPU 实现，通过 `WorkTileScheduler` 按瓦片调度，管理 SoA 布局的波前积分器状态、内核排队和着色器排序

### AdaptiveSampling

自适应采样参数与工具类：

- `align_samples()`：对齐采样数，确保所有设备在相同的样本点执行滤波
- `need_filter()`：判断是否需要在当前样本执行收敛检测
- 参数包括：`adaptive_step`（检测间隔）、`min_samples`（最小采样数）、`threshold`（收敛阈值）

### Denoiser

降噪器抽象基类，定义统一的降噪接口：

- `create()`：工厂方法，根据设备能力和参数创建合适的降噪器实现
- `denoise_buffer()`：对整个渲染缓冲区执行降噪（纯虚函数）
- `load_kernels()`：加载降噪所需的内核
- `automatic_viewport_denoiser_type()`：推荐的 viewport 降噪器类型

降噪器继承层次：
```
Denoiser (抽象基类)
├── OIDNDenoiser        — CPU 端 OpenImageDenoise 降噪
└── DenoiserGPU (GPU 基类)
    ├── OIDNDenoiserGPU — GPU 端 OpenImageDenoise 降噪
    └── OptiXDenoiser   — NVIDIA OptiX 降噪
```

### PassAccessor

渲染通道访问器，负责从渲染缓冲区读取各类通道（深度、法线、运动矢量、Cryptomatte 等）并转换为目标格式：

- 支持 float 和 half4 目标格式
- 处理曝光、采样数缩放、Shadow Catcher 合成等
- 两个实现：`PassAccessorCPU`（CPU 端胶片转换）和 `PassAccessorGPU`（GPU 端内核转换）

### WorkBalancer

多设备工作负载均衡器：

- `work_balance_do_initial()`：初始渲染时的均匀分配
- `work_balance_do_rebalance()`：根据已收集的时间和设备占用率统计信息重新分配工作

`WorkBalanceInfo` 记录每个设备的耗时、占用率和归一化权重。

### WorkTileScheduler

工作瓦片调度器，将大瓦片（big tile）分割为适合设备执行的小工作瓦片。主要用于 GPU 路径追踪：

- 根据图像尺寸、最大路径状态数和乱序距离计算最优瓦片大小
- 通过 `get_work()` 接口向设备提供下一个待渲染的工作瓦片
- 支持加速光线追踪（RT Core）的瓦片大小优化

### ShaderEval

着色器离线求值器，在渲染前对场景进行预处理：

- `SHADER_EVAL_DISPLACE`：位移贴图求值
- `SHADER_EVAL_BACKGROUND`：背景光求值
- `SHADER_EVAL_CURVE_SHADOW_TRANSPARENCY`：曲线阴影透明度
- `SHADER_EVAL_VOLUME_DENSITY`：体积密度求值

### GuidingParams

路径引导参数，控制基于 OpenPGL 的路径引导行为：

- `use_surface_guiding` / `use_volume_guiding`：表面/体积引导开关
- `type`：引导分布类型（如视差感知 VMM）
- `sampling_type`：方向采样策略（如乘积 MIS）
- `training_samples`：训练样本数
- `roughness_threshold`：粗糙度阈值（低于此值才启用引导）

### PathTraceDisplay

`DisplayDriver` 的包装器，添加线程安全、状态跟踪和错误检查：

- 管理纹理更新流程（`update_begin` / `update_end`）
- 支持 CPU 像素复制和 GPU 图形互操作（Graphics Interop）两种更新路径
- 跟踪纹理状态（是否过期、尺寸等）

### PathTraceTile

`OutputDriver::Tile` 接口的实现，作为路径追踪器与输出驱动之间的桥梁，允许输出驱动按名称访问渲染通道像素数据。

## 模块架构

### 渲染调度流程

```
Session::thread_render()
  └─> RenderScheduler::get_render_work()   // 调度器决定本轮工作内容
       └─> PathTrace::render(render_work)   // 主控类执行渲染管线
            │
            ├─ init_render_buffers()         // 1. 初始化/清零渲染缓冲区
            │
            ├─ path_trace()                  // 2. 路径追踪采样
            │   └─ PathTraceWork::render_samples()
            │       ├─ [CPU] 逐像素，TBB 并行完整路径
            │       └─ [GPU] 波前式，按工作瓦片调度内核
            │
            ├─ adaptive_sample()             // 3. 自适应采样收敛检测与滤波
            │   └─ PathTraceWork::adaptive_sampling_converge_filter_count_active()
            │
            ├─ denoise()                     // 4. 降噪（可选）
            │   └─ Denoiser::denoise_buffer()
            │       ├─ OIDNDenoiser (CPU)
            │       ├─ OIDNDenoiserGPU (GPU OIDN)
            │       └─ OptiXDenoiser (GPU OptiX)
            │
            ├─ cryptomatte_postprocess()     // 5. Cryptomatte 后处理
            │
            ├─ update_display()              // 6. 更新显示纹理
            │   └─ PathTraceWork::copy_to_display()
            │       ├─ 朴素模式：胶片转换 → CPU 复制 → 上传纹理
            │       └─ 互操作模式：直接 GPU→GPU 写入纹理
            │
            ├─ rebalance()                   // 7. 多设备工作再平衡
            │   └─ WorkBalancer::work_balance_do_rebalance()
            │
            └─ write_tile_buffer()           // 8. 写入瓦片到磁盘或回调
```

### GPU 波前渲染内部流程

```
PathTraceWorkGPU::render_samples()
  ├─ enqueue_reset()                      // 重置积分器状态
  ├─ enqueue_work_tiles()                 // 从 WorkTileScheduler 获取瓦片并入队
  └─ while (有活跃路径):
       ├─ get_most_queued_kernel()        // 找到队列中路径最多的内核
       ├─ compute_sorted_queued_paths()   // 着色器排序（提升缓存命中率）
       └─ enqueue_path_iteration()        // 执行一步积分内核
```

## 依赖关系

### 上游依赖（本模块依赖）

| 模块 | 用途 |
|------|------|
| `device/` | 设备抽象层：`Device`、`DeviceQueue`、`DeviceScene`、`device_memory`、`DenoiseParams` |
| `scene/` | 场景数据：`Film`（胶片配置）、`Pass`（渲染通道定义） |
| `session/` | 会话支持：`BufferParams`、`RenderBuffers`、`DisplayDriver`、`OutputDriver`、`TileManager` |
| `kernel/` | 内核类型定义：`KernelWorkTile`、`KernelFilmConvert`、`IntegratorState` |
| `util/` | 工具库：线程、向量、字符串、半精度浮点等 |
| OpenPGL | 外部库（可选）：路径引导的场和采样分布 |
| OpenImageDenoise | 外部库（可选）：OIDN 降噪 |
| OptiX | 外部库（可选）：OptiX 降噪 |

### 下游依赖（依赖本模块）

| 模块 | 用途 |
|------|------|
| `session/` | `Session` 创建和驱动 `PathTrace`、`RenderScheduler` |
| `blender/` | Blender 集成层通过 `Session` 间接使用积分器 |
| `app/` | 命令行渲染应用通过 `Session` 间接使用积分器 |

## 关键算法与实现细节

### 自适应采样策略

自适应采样通过定期检查像素是否已收敛来跳过不必要的采样。关键设计：

1. **对齐机制**：`AdaptiveSampling::align_samples()` 确保所有设备在完全相同的样本点执行滤波检查，无论各设备的性能差异。这保证了多设备渲染结果的一致性。
2. **渐进噪声阈值**（Progressive Noise Floor）：RenderScheduler 可以先用较高阈值快速让大部分像素收敛，然后逐步降低阈值精细化剩余区域，保持画面均匀的噪声水平。
3. **空闲重调度**：当大多数像素已收敛、设备大量空闲时，`render_work_reschedule_on_idle()` 可以触发更低阈值的重新调度。

### viewport 交互式渲染优化

`RenderScheduler` 实现了多种启发式策略以保证交互流畅性：

1. **分辨率缩放**：在 viewport 导航时自动降低渲染分辨率（`resolution_divider`），基于首次渲染的时间估算计算最优缩放比。
2. **渐进式显示更新**：低采样时频繁更新显示，高采样时降低更新频率以提升设备利用率。
3. **延迟降噪**：在快速导航时可跳过降噪步骤，避免不必要的开销。

### 多设备工作分配

1. 大瓦片（big tile）在所有设备间按权重切片，每个设备渲染自己的切片。
2. `WorkBalancer` 根据各设备的实际渲染时间和占用率计算权重。
3. 权重变化时触发再平衡，重新分配切片大小。
4. 降噪在单独的设备上对合并后的完整大瓦片执行。

### GPU 波前积分器状态管理

GPU 实现使用 SoA (Structure of Arrays) 布局存储积分器状态，并通过以下机制优化性能：

- **着色器排序**：将待处理路径按材质排序，提升 GPU warp 内的执行一致性
- **路径压缩**：终止的路径被压缩移除，保持活跃路径连续
- **分区排序**：使用分区键偏移实现高效的前缀和排序

## 参见

- `src/session/` — 渲染会话管理，创建和驱动积分器
- `src/device/` — 设备抽象层，提供计算和降噪设备
- `src/scene/` — 场景描述，提供 Film 和 Pass 配置
- `src/kernel/` — GPU/CPU 内核实现，积分器在设备端的实际执行代码
