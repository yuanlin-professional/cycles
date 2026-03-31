# kernel/film - 胶片/成像缓冲区

## 概述

`kernel/film` 模块负责 Cycles 路径追踪渲染器中渲染结果的累积、存储和读取。它管理渲染缓冲区（render buffer）中的像素数据写入（路径追踪累积）和读取（显示/输出），支持多种渲染通道（render passes）：组合通道、光照通道、数据通道、降噪通道、Cryptomatte、AOV 自定义通道等。

该模块还实现了自适应采样（Adaptive Sampling）的收敛检测逻辑，以及阴影捕获器（Shadow Catcher）的合成计算。GPU 渲染时使用原子操作保证多线程写入的正确性。

## 目录结构

| 文件名 | 类型 | 说明 |
|--------|------|------|
| `write.h` | 写入核心 | 渲染缓冲区写入函数：像素定位、float/float3/float4/Spectrum 的原子累积写入 |
| `read.h` | 读取核心 | 渲染缓冲区读取与格式转换：缩放、曝光、透明度到 Alpha 转换、阴影捕获器合成、各通道类型读取 |
| `light_passes.h` | 光照通道 | BSDF 评估结果分离（漫射/光泽）、光照通道累积（直接/间接漫射、光泽、透射）、辐射写入 |
| `data_passes.h` | 数据通道 | 数据型渲染通道写入：法线、UV、运动向量、对象/材质索引、深度、雾 |
| `denoising_passes.h` | 降噪通道 | 降噪数据写入：降噪法线、降噪反照率（albedo） |
| `adaptive_sampling.h` | 自适应采样 | 自适应采样的收敛判定：像素方差检测、采样停止/继续决策 |
| `cryptomatte_passes.h` | Cryptomatte | Cryptomatte 通道写入：对象 ID 和资产 ID 的排序累积 |
| `aov_passes.h` | AOV 通道 | 任意输出变量（AOV）通道写入，支持颜色和标量类型 |
| `volume_guiding_denoise.h` | 体积引导降噪 | 体积渲染相关的降噪通道写入 |

## 核心类与数据结构

- **`KernelFilmConvert`**：胶片转换参数结构体，用于读取时的通道映射和缩放。包含各通道偏移量、曝光值、缩放因子、组件数等。
- **`BsdfEval`**：BSDF 评估结果，按漫射（`diffuse`）和光泽（`glossy`）分量分离存储，另有 `sum` 字段存储总和。用于将积分结果正确累积到对应的光照通道。
- **渲染缓冲区布局**：每个像素在缓冲区中占 `pass_stride` 个 float，各通道通过偏移量（`pass_combined`、`pass_depth` 等）定位。`PASS_UNUSED` 标记未启用的通道。

## 内核函数入口

### 写入函数（路径追踪阶段）
- **`film_pass_pixel_render_buffer()`**：根据积分器状态或像素坐标计算渲染缓冲区指针偏移。
- **`film_write_pass_float()` / `film_write_pass_float3()` / `film_write_pass_float4()`**：向缓冲区累积写入标量/向量/四元数据，GPU 上使用 `atomic_add_and_fetch_float` 原子操作。
- **`film_write_pass_spectrum()`**：写入光谱数据到缓冲区。
- **`bsdf_eval_init()` / `bsdf_eval_accum()`**：初始化/累积 BSDF 评估结果，自动按闭包类型分离漫射和光泽分量。

### 读取函数（显示/输出阶段）
- **`film_get_pass_pixel_combined()`**：读取组合通道，执行缩放、曝光和透明度到 Alpha 的转换。
- **`film_get_pass_pixel_depth()` / `film_get_pass_pixel_mist()`**：读取深度/雾通道。
- **`film_get_pass_pixel_light_path()`**：读取光照通道，支持间接光叠加和颜色除法。
- **`film_get_pass_pixel_motion()`**：读取运动向量通道，使用运动权重归一化。
- **`film_get_pass_pixel_cryptomatte()`**：读取 Cryptomatte 通道（ID + 权重交替存储）。
- **`film_get_pass_pixel_shadow_catcher()`**：读取阴影捕获器通道，执行组合/阴影捕获器的比值计算。

### 自适应采样
- **`film_need_sample_pixel()`**：检查像素是否已收敛，决定是否跳过采样。
- **`film_adaptive_sampling_convergence_check()`**：执行收敛判定，比较像素方差与阈值。

## GPU 兼容性

该模块对 GPU 有特殊适配：

- **原子写入**：GPU 上通过 `__ATOMIC_PASS_WRITE__` 宏启用原子浮点累加（`atomic_add_and_fetch_float`），因为多个路径可能同时写入同一像素。CPU 上使用普通加法。
- **缓冲区偏移**：使用 `uint64_t` 计算缓冲区偏移量，避免大分辨率/高通道数时的 32 位溢出。
- **内存布局**：渲染缓冲区使用线性布局，每像素 `pass_stride` 个 float，对 GPU 的内存合并（coalescing）友好。

## 依赖关系

### 上游依赖（本模块依赖）
- `kernel/globals.h`：全局内核数据（`kernel_data.film` 胶片参数）
- `kernel/types.h`：基础类型定义（`PASS_UNUSED` 等常量）
- `kernel/integrator/state.h`：积分器状态（获取 `render_pixel_index`）
- `kernel/integrator/shadow_catcher.h`：阴影捕获器状态判定
- `kernel/sample/pattern.h`：采样模式（自适应采样使用）
- `kernel/util/colorspace.h`：色彩空间转换工具
- `util/atomic.h`：原子操作（GPU 写入）
- `util/color.h`：颜色工具函数
- `util/types_rgbe.h`：RGBE 编码/解码

### 下游依赖（依赖本模块）
- `kernel/integrator/shade_surface.h`：表面着色后将光照结果写入对应通道
- `kernel/integrator/shade_volume.h`：体积着色后写入光照通道
- `kernel/integrator/shade_background.h`：背景着色后写入通道
- `kernel/integrator/shade_shadow.h`：阴影评估后写入阴影通道
- `kernel/integrator/init_from_camera.h`：初始化时检查自适应采样收敛状态
- `kernel/integrator/state_flow.h`：路径结束时写入光学深度

## 关键算法与实现细节

1. **原子浮点累加**：GPU 渲染时，多个线程可能同时写入同一像素的同一通道。模块使用 `atomic_add_and_fetch_float()` 实现无锁的浮点原子累加，保证数据正确性但可能影响性能。CPU 渲染时直接使用非原子加法。

2. **BSDF 分量分离**：`bsdf_eval_init()` 和 `bsdf_eval_accum()` 根据闭包类型（`CLOSURE_IS_BSDF_DIFFUSE`、`CLOSURE_IS_BSDF_GLOSSY`、`CLOSURE_IS_GLASS`）自动将评估结果分离到漫射和光泽通道。玻璃闭包根据出射方向与法线的关系动态归类为光泽或透射。

3. **自适应采样收敛检测**：使用辅助缓冲区存储像素颜色的累积和（`I`）和半方差辅助数据（`A`）。通过比较归一化方差与用户设定的阈值，决定像素是否已充分收敛。收敛的像素在辅助缓冲区的 `w` 分量标记为非零值，后续采样时跳过。

4. **阴影捕获器合成**：
   - 阴影值通过 `combined_no_matte / shadow_catcher` 除法计算得到。
   - 去噪模式下直接使用缩放后的阴影捕获器通道值。
   - 边缘抗锯齿通过在白色背景上使用组合通道透明度进行 alpha-over 混合。
   - 遮罩物体（matte object）的贡献从组合通道中减去以避免除法伪影。

5. **每像素采样数跟踪**：`pass_sample_count` 通道以 `uint`（通过 `__float_as_uint` 编码）记录每像素实际采样数，用于自适应采样中的归一化缩放。

6. **RGBE 编码**：部分通道（如路径引导数据）使用 RGBE 紧凑编码，通过 `rgbe_to_rgb()` 解码读取，节省缓冲区内存。

## 参见

- `src/kernel/integrator/` - 积分器内核（写入渲染通道的调用方）
- `src/render/film.cpp` - 主机端胶片配置与通道布局
- `src/render/buffers.cpp` - 渲染缓冲区内存管理
- `src/kernel/integrator/shadow_catcher.h` - 阴影捕获器路径分裂逻辑
