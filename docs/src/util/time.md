# time.h - 高精度计时与作用域计时器工具集

## 概述

`time.h` 提供了 Cycles 渲染器中使用的时间测量工具集，包括高精度墙钟时间获取、线程休眠、快速硬件计时器（基于 RDTSCP）以及多种作用域自动计时器类。这些工具广泛用于渲染性能分析、进度跟踪、任务调度等场景。此外还提供了时间字符串的人类可读格式转换功能，兼容 Blender 元数据格式。

## 核心函数/类

### 全局函数

#### `time_dt()`
- **签名**: `double time_dt()`
- **功能**: 返回当前时间（秒），双精度浮点，具有高精度。实现在 `time.cpp` 中，通常基于平台特定的高精度时钟

#### `time_sleep(t)`
- **签名**: `void time_sleep(const double t)`
- **功能**: 使当前线程休眠指定秒数

#### `time_fast_tick(last_cpu)`
- **签名**: `uint64_t time_fast_tick(uint32_t *last_cpu)`
- **功能**: 快速获取硬件时间戳计数器值。在 x86 平台上使用 `RDTSCP` 指令，同时返回 CPU 核心编号（通过 `last_cpu` 输出参数），以便检测线程是否在测量期间发生了 CPU 迁移
- **用途**: 适用于对开销敏感的高频微基准测试

#### `time_fast_frequency()`
- **签名**: `uint64_t time_fast_frequency()`
- **功能**: 返回快速计时器的频率（每秒 tick 数），用于将 `time_fast_tick()` 的返回值转换为实际时间

#### `time_human_readable_from_seconds(seconds)`
- **签名**: `string time_human_readable_from_seconds(const double seconds)`
- **功能**: 将秒数转换为人类可读的时间字符串，格式兼容 Blender 元数据

#### `time_human_readable_to_seconds(time_string)`
- **签名**: `double time_human_readable_to_seconds(const string &time_string)`
- **功能**: 将人类可读的时间字符串解析为秒数

### `scoped_timer`
- **功能**: 基于 RAII 的作用域计时器，在构造时记录起始时间，在析构时将经过的时间写入指定的 `double` 指针
- **构造函数**: `explicit scoped_timer(double *value = nullptr)` — 可选地接收一个指针，析构时自动写入耗时
- **方法**:
  - `get_start()` — 返回计时器的起始时间戳
  - `get_time()` — 返回从构造至今的经过时间（秒）

### `fast_timer`
- **功能**: 基于硬件时间戳的高性能计时器，适用于对测量开销极度敏感的场景
- **构造函数**: 初始化时记录当前 CPU 编号和 tick 值
- **方法**:
  - `lap(delta)` — 记录一次计时间隔，通过输出参数 `delta` 返回自上次 lap 以来的 tick 差值。返回值为 `bool`，指示两次 lap 是否在同一 CPU 核心上执行（`true` 表示测量结果可靠，`false` 表示可能因 CPU 迁移导致结果不准确）
- **成员**:
  - `last_cpu` (`uint32_t`) — 上次测量时的 CPU 核心编号
  - `last_value` (`uint64_t`) — 上次测量的 tick 值

### `scoped_callback_timer`
- **功能**: 带回调函数的作用域计时器，在析构时通过回调函数传递经过的时间
- **构造函数**: `explicit scoped_callback_timer(callback_type cb)` — 接收 `std::function<void(double)>` 类型的回调
- **行为**: 析构时调用 `cb(timer.get_time())`，将耗时传递给回调处理

## 依赖关系

- **内部头文件**:
  - `util/string.h` — 字符串类型定义（`string` 类型别名）
- **C++ 标准库**:
  - `<functional>` — `std::function` 用于 `scoped_callback_timer` 的回调类型
- **被引用**: 被 Cycles 中大量模块引用，包括：
  - `src/util/time.cpp` — 函数实现
  - `src/util/task.cpp` — 任务系统性能追踪
  - `src/util/progress.h` — 渲染进度计时
  - `src/session/session.cpp` — 渲染会话计时
  - `src/integrator/path_trace.cpp` — 路径追踪性能监控
  - `src/integrator/render_scheduler.cpp` — 渲染调度器时间管理
  - `src/bvh/build.cpp` — BVH 构建计时
  - `src/scene/` 下多个文件 — 场景更新计时
  - `src/test/util_time_test.cpp` — 单元测试

## 实现细节

1. **RDTSCP 与 CPU 迁移检测**: `fast_timer` 利用 x86 的 `RDTSCP` 指令同时获取时间戳和 CPU ID。如果线程在两次测量间被操作系统调度到不同核心，不同核心可能有不同的时钟状态（尤其在启用 CPU 频率缩放时），`lap()` 通过比较 CPU ID 来标记这种不可靠的测量结果
2. **scoped_timer 的安全性**: 当构造时传入 `nullptr`，析构时不会写入任何值，可安全用于仅需 `get_time()` 手动查询的场景
3. **回调计时器模式**: `scoped_callback_timer` 内部组合了一个 `scoped_timer`，将直接指针写入模式替换为更灵活的回调模式，适用于需要在计时结束时执行复杂逻辑（如日志记录、统计聚合）的场景

## 关联文件

- `src/util/time.cpp` — `time_dt()`、`time_sleep()`、`time_fast_tick()` 等函数的平台相关实现
- `src/util/string.h` — 字符串类型定义
- `src/test/util_time_test.cpp` — 时间工具单元测试
