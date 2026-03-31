# tbb.h - Intel TBB 并行框架封装

## 概述

`tbb.h` 是 Cycles 对 Intel Threading Building Blocks (TBB) 库的统一封装头文件。它在 Windows 上确保 `<windows.h>` 以精简模式优先包含（避免 TBB 间接引入未定义 `WIN32_LEAN_AND_MEAN` 的版本），并将常用 TBB 并行原语引入 `ccl` 命名空间。同时提供浮点环境捕获和并行取消的辅助函数。

## 类与结构体

无自定义类或结构体。

## 核心函数/宏定义

### 引入的 TBB 类型（using 声明）

| 名称 | TBB 原始类型 | 说明 |
|------|-------------|------|
| `blocked_range` | `tbb::blocked_range` | 一维索引范围 |
| `blocked_range2d` | `tbb::blocked_range2d` | 二维索引范围 |
| `blocked_range3d` | `tbb::blocked_range3d` | 三维索引范围 |
| `enumerable_thread_specific` | `tbb::enumerable_thread_specific` | 线程局部存储容器 |
| `parallel_for` | `tbb::parallel_for` | 并行 for 循环 |
| `parallel_for_each` | `tbb::parallel_for_each` | 并行遍历容器 |
| `parallel_reduce` | `tbb::parallel_reduce` | 并行归约 |

### 宏定义

| 宏 | 说明 |
|----|------|
| `WITH_TBB_GLOBAL_CONTROL` | TBB 接口版本 >= 10 时定义，启用全局线程数控制 |
| `TBB_PREVIEW_GLOBAL_CONTROL` | 启用 TBB 预览版全局控制 API |

### 辅助函数

| 函数 | 说明 |
|------|------|
| `thread_capture_fp_settings()` | 捕获当前线程的浮点环境设置到 TBB 任务组上下文中，确保工作线程继承主线程的浮点模式 |
| `parallel_for_cancel()` | 取消当前正在执行的 `parallel_for` 任务组 |

两个函数均根据 `TBB_INTERFACE_VERSION_MAJOR` 使用不同 API：
- TBB >= 12：使用 `tbb::task::current_context()`
- TBB < 12：使用 `tbb::task::self().group()`

## 依赖关系

- **内部头文件**: `util/windows.h`（仅 Windows 平台）
- **外部依赖**: `<tbb/blocked_range2d.h>`, `<tbb/blocked_range3d.h>`, `<tbb/enumerable_thread_specific.h>`, `<tbb/parallel_for.h>`, `<tbb/parallel_for_each.h>`, `<tbb/parallel_reduce.h>`, `<tbb/task_arena.h>`, `<tbb/task_group.h>`, `<tbb/global_control.h>`
- **被引用**: `util/task.h`, `subd/dice.cpp`, `scene/object.cpp`, `scene/hair.cpp`, `scene/camera.cpp`, `integrator/shader_eval.cpp`, `integrator/path_trace_work_cpu.cpp`, `integrator/path_trace.cpp`, `integrator/pass_accessor_cpu.cpp`, `app/cycles_precompute.cpp`

## 实现细节

1. **Windows 头文件顺序**：TBB 头文件会间接包含 `<windows.h>`。为确保 `NOGDI`、`NOMINMAX`、`WIN32_LEAN_AND_MEAN` 等宏被预先定义，本文件在 `_WIN32` 平台上先包含 `util/windows.h`。

2. **浮点环境传播**：渲染器可能修改浮点控制标志（如 Flush-to-Zero）。`thread_capture_fp_settings()` 确保 TBB 工作线程使用与提交任务的线程相同的浮点设置，避免渲染结果不一致。

3. **TBB 版本兼容**：文件中多处使用 `TBB_INTERFACE_VERSION_MAJOR` 进行版本分支，保证对 TBB 10/12+ 版本的兼容。

## 关联文件

- `util/windows.h` - Windows 平台头文件预处理
- `util/task.h` / `util/task.cpp` - 使用 TBB 的任务调度系统
- `util/thread.h` - 线程基础原语（使用 `tbb::spin_mutex`）
