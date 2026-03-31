# profiling.h / profiling.cpp - 渲染内核性能采样分析器

## 概述

`profiling.h` 和 `profiling.cpp` 实现了 Cycles 的内核级性能分析器，通过周期性采样（1ms 间隔）统计渲染工作线程在各阶段（光线设置、交点计算、着色等）的时间分布，以及各着色器和物体的命中频率。采样式设计开销极低，适合在实际渲染过程中使用。

## 类与结构体

### `ProfilingEvent` 枚举

定义了渲染内核中所有可采样的事件阶段：

| 事件 | 说明 |
|------|------|
| `PROFILING_UNKNOWN` | 未知/空闲状态 |
| `PROFILING_RAY_SETUP` | 光线初始化 |
| `PROFILING_INTERSECT_CLOSEST` | 最近交点计算 |
| `PROFILING_INTERSECT_SUBSURFACE` | 次表面散射交点 |
| `PROFILING_INTERSECT_SHADOW` | 阴影光线交点 |
| `PROFILING_INTERSECT_VOLUME_STACK` | 体积栈交点 |
| `PROFILING_INTERSECT_DEDICATED_LIGHT` | 专用光源交点 |
| `PROFILING_SHADE_SURFACE_*` | 表面着色各阶段（设置、求值、直接光、间接光、AO、通道） |
| `PROFILING_SHADE_VOLUME_*` | 体积着色各阶段 |
| `PROFILING_SHADE_SHADOW_*` | 阴影着色各阶段 |
| `PROFILING_SHADE_LIGHT_*` | 光源着色各阶段 |
| `PROFILING_SHADE_DEDICATED_LIGHT` | 专用光源着色 |
| `PROFILING_NUM_EVENTS` | 事件总数（用于数组大小） |

### `ProfilingState`

每个工作线程持有的采样状态，由工作线程持续更新，由分析器线程周期性读取。

| 字段 | 说明 |
|------|------|
| `volatile uint32_t event` | 当前执行阶段 |
| `volatile int32_t shader` | 当前着色器 ID（-1 表示无） |
| `volatile int32_t object` | 当前物体 ID（-1 表示无） |
| `volatile bool active` | 是否处于活跃状态 |
| `vector<uint64_t> shader_hits` | 线程局部着色器命中计数 |
| `vector<uint64_t> object_hits` | 线程局部物体命中计数 |

**注意**：字段使用 `volatile` 而非原子类型——因为在 x86 上对齐的 32 位读写本身是原子的，偶尔读到中间状态最多导致一次错误采样，对统计结果影响可忽略。

### `Profiler`

核心分析器类，管理采样线程和数据聚合。

| 方法 | 说明 |
|------|------|
| `reset(int num_shaders, int num_objects)` | 重置所有计数器并调整大小 |
| `start()` | 启动采样工作线程 |
| `stop()` | 停止采样工作线程 |
| `add_state(ProfilingState *state)` | 注册工作线程的采样状态 |
| `remove_state(ProfilingState *state)` | 注销并合并线程局部命中计数 |
| `get_event(ProfilingEvent event)` | 获取某事件的采样次数 |
| `get_shader(int shader, uint64_t &samples, uint64_t &hits)` | 获取着色器的采样和命中数据 |
| `get_object(int object, uint64_t &samples, uint64_t &hits)` | 获取物体的采样和命中数据 |
| `bool active() const` | 分析器是否正在运行 |

**保护成员：**
- `event_samples` / `shader_samples` / `object_samples` - 采样计数向量
- `shader_hits` / `object_hits` - 全局命中聚合向量
- `do_stop_worker` - 采样线程停止标志
- `worker` - 采样线程指针
- `mutex` - 保护 states 列表
- `states` - 已注册的 ProfilingState 指针列表

### `ProfilingHelper`

RAII 风格的性能事件标记器。构造时设置当前事件，析构时恢复前一个事件。

| 方法 | 说明 |
|------|------|
| `ProfilingHelper(ProfilingState *state, ProfilingEvent event)` | 保存旧事件并设置新事件 |
| `~ProfilingHelper()` | 恢复旧事件 |
| `set_event(ProfilingEvent event)` | 中途更改事件 |

### `ProfilingWithShaderHelper`

继承自 `ProfilingHelper`，额外支持着色器/物体跟踪。

| 方法 | 说明 |
|------|------|
| `set_shader(int object, int shader)` | 设置当前着色器和物体 ID，并递增命中计数 |
| `~ProfilingWithShaderHelper()` | 析构时重置 shader 和 object 为 -1 |

## 核心函数/宏定义

无额外宏定义。

## 依赖关系

- **内部头文件**: `util/thread.h`, `util/unique_ptr.h`, `util/vector.h`
- **外部依赖**: `<cassert>`, `<chrono>`, `<thread>`, `<algorithm>`
- **被引用**: `kernel/device/cpu/globals.cpp`, `device/device.h`

## 实现细节

1. **采样策略**：采样线程以 1ms 间隔唤醒，读取所有已注册 `ProfilingState` 的当前事件/着色器/物体，递增对应计数器。使用绝对时间点计算下次唤醒时间（`sleep_until`），避免因 `sleep_for` 的累计漂移。

2. **线程安全设计**：`ProfilingState` 的读写不需要锁保护（基于 x86 原子性保证）。`states` 列表的添加/移除使用 `mutex` 保护。数据查询函数（`get_event`/`get_shader`/`get_object`）要求分析器已停止（`assert(worker == nullptr)`）。

3. **命中计数合并**：每个线程有独立的 `shader_hits`/`object_hits` 计数器，在 `remove_state()` 时合并到全局计数器，避免写竞争。

4. **生命周期管理**：`reset()` 在重置计数器前自动停止采样线程，重置后若原先在运行则重新启动。析构函数断言 `worker == nullptr`，要求外部先调用 `stop()`。

5. **RAII 事件标记**：`ProfilingHelper` 和 `ProfilingWithShaderHelper` 允许在代码作用域中自动标记性能事件，离开作用域时自动恢复，确保事件状态一致性。

## 关联文件

- `util/thread.h` - 提供线程类和同步原语
- `device/device.h` - 设备层持有 Profiler 实例
- `kernel/device/cpu/globals.cpp` - CPU 内核中注册 ProfilingState
