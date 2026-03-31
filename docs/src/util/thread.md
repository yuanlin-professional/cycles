# thread.h / thread.cpp - 线程基础设施与同步原语

## 概述

`thread.h` 和 `thread.cpp` 为 Cycles 渲染器提供跨平台线程抽象层。该模块封装了自定义线程类（支持 macOS/musl 上的自定义栈大小）、互斥锁、条件变量以及自旋锁等同步原语。在 macOS 和非 glibc Linux 系统上使用 POSIX pthread，在其他平台上使用 `std::thread`。

## 类与结构体

### `thread`

自定义线程实现，类似于 `std::thread`，但支持在 macOS 上设置自定义栈大小（2MB，与 GLIBC 一致）。

| 成员 | 说明 |
|------|------|
| `thread(std::function<void()> run_cb)` | 构造函数，传入线程执行回调并立即启动线程 |
| `~thread()` | 析构函数，若未 join 则自动调用 `join()` |
| `static void *run(void *arg)` | 静态入口函数，内部调用 `run_cb_()` |
| `bool join()` | 等待线程结束，返回是否成功 |
| `run_cb_` | 线程回调函数 (`std::function<void()>`) |
| `joined_` | 标记是否已 join |

**平台分支：**
- macOS / 非 glibc Linux：使用 `pthread_t`，通过 `pthread_attr_setstacksize` 设置 2MB 栈
- 其他平台（Windows/glibc Linux）：使用 `std::thread`

### `thread_scoped_spin_lock`

RAII 风格的自旋锁守卫，构造时加锁，析构时解锁。

| 成员 | 说明 |
|------|------|
| `thread_scoped_spin_lock(thread_spin_lock &lock)` | 构造并加锁 |
| `~thread_scoped_spin_lock()` | 析构并解锁 |

## 核心函数/宏定义

### 类型别名

| 别名 | 实际类型 | 说明 |
|------|----------|------|
| `thread_mutex` | `std::mutex` | 互斥锁 |
| `thread_scoped_lock` | `std::unique_lock<std::mutex>` | RAII 互斥锁守卫 |
| `thread_condition_variable` | `std::condition_variable` | 条件变量 |
| `thread_spin_lock` | `tbb::spin_mutex` | TBB 自旋锁 |

## 依赖关系

- **内部头文件**: `util/windows.h`（仅 Windows）
- **外部依赖**: `<tbb/spin_mutex.h>`, `<pthread.h>`（macOS/musl）, `<thread>`（其他平台）, `<mutex>`, `<condition_variable>`, `<functional>`
- **被引用**: `util/task.h`, `util/semaphore.h`, `util/profiling.h`, `util/progress.h`, `session/session.h`, `scene/scene.h`, `scene/shader.h`, `scene/light.h`, `scene/image.h`, `scene/colorspace.cpp`, `integrator/path_trace_display.h`, `integrator/path_trace.h`, `integrator/denoiser_oidn.h`, `device/device.h`, `graph/node_type.h` 等

## 实现细节

1. **栈大小设置**：macOS 默认线程栈仅 512KB，不足以支撑 Embree 等库的需求。构造函数中通过 `pthread_attr_setstacksize` 将栈大小设为 2MB，与 GLIBC 默认值保持一致。

2. **join 安全性**：析构函数会检查 `joined_` 标志，未 join 的线程会在析构时自动 join，避免 `std::thread` 析构未 join 线程时的终止行为。

3. **自旋锁选择**：使用 TBB 的 `spin_mutex` 而非 `util/tbb.h` 中的完整 TBB 头文件，是因为部分 TBB 功能需要 RTTI，而 OSL 内核编译时 RTTI 被禁用。

4. **异常处理**：在非 pthread 路径下，`join()` 使用 try-catch 捕获 `std::system_error`，保证不会因异常传播导致崩溃。

## 关联文件

- `util/task.h` / `util/task.cpp` - 基于线程的任务调度系统
- `util/tbb.h` - TBB 并行框架封装
- `util/semaphore.h` - 基于本模块原语实现的计数信号量
- `util/profiling.h` - 使用本模块线程实现性能采样
