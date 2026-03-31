# task.h / task.cpp - 任务池与调度系统

## 概述

`task.h` 和 `task.cpp` 实现了 Cycles 的任务调度系统，提供三个核心组件：`TaskPool`（基于 TBB 的并行任务池）、`TaskScheduler`（全局线程调度器）和 `DedicatedTaskPool`（单线程专用任务池）。任务函数通过 `std::function<void()>` 传递，支持 lambda 表达式和 `std::bind`。

## 类与结构体

### `TaskPool`

基于 TBB `task_group` 的并行任务池，支持等待所有任务完成或提前取消。

| 成员/方法 | 说明 |
|-----------|------|
| `push(TaskRunFunction &&task)` | 提交任务到池中（通过 `tbb_group.run` 执行） |
| `wait_work(Summary *stats = nullptr)` | 阻塞等待所有任务完成，可选收集统计信息 |
| `cancel()` | 取消所有待执行任务并等待正在执行的任务完成 |
| `static canceled()` | 供工作线程检测当前任务池是否已取消 |

#### `TaskPool::Summary`

| 字段 | 说明 |
|------|------|
| `double time_total` | 处理所有任务的总耗时 |
| `int num_tasks_handled` | 已处理的任务总数 |
| `string full_report() const` | 生成多行统计报告字符串 |

### `TaskScheduler`

全局单例任务调度器，管理 TBB 线程池。支持多个 Cycles 实例共享线程池（引用计数机制）。

| 方法 | 说明 |
|------|------|
| `static init(int num_threads = 0)` | 初始化调度器；`num_threads > 0` 时覆盖 TBB 默认线程数 |
| `static exit()` | 减少引用计数，计数归零时释放全局控制 |
| `static free_memory()` | 释放内存（需在 users == 0 时调用） |
| `static max_concurrency()` | 返回最大并发线程数 |

**静态成员：**
- `mutex` - 保护初始化/销毁的互斥锁
- `users` - 引用计数
- `active_num_threads` - 当前活跃线程数
- `global_control` - TBB 全局线程数控制（TBB >= 10 时可用）

### `DedicatedTaskPool`

单线程专用任务池，启动一个专用工作线程顺序执行所有任务。

| 方法 | 说明 |
|------|------|
| `push(TaskRunFunction &&run, bool front = false)` | 提交任务，`front=true` 时插入队列头部 |
| `wait()` | 等待所有已提交任务完成 |
| `cancel()` | 取消待执行任务（不终止工作线程） |
| `canceled()` | 检测是否已取消 |

**内部机制：**
- 使用 `list<TaskRunFunction>` 作为任务队列
- 通过 `num_mutex` / `num_cond` 实现任务计数同步
- 通过 `queue_mutex` / `queue_cond` 实现任务队列的线程安全访问
- 工作线程在 `thread_run()` 中循环 pop 并执行任务

## 核心函数/宏定义

| 定义 | 说明 |
|------|------|
| `TaskRunFunction` | `std::function<void()>` 的类型别名 |

## 依赖关系

- **内部头文件**: `util/list.h`, `util/string.h`, `util/tbb.h`, `util/thread.h`, `util/unique_ptr.h`, `util/log.h`, `util/time.h`
- **被引用**: `session/session.cpp`, `session/denoising.cpp`, `scene/svm.cpp`, `scene/integrator.cpp`, `scene/light_tree.h`, `scene/geometry.cpp`, `scene/image.cpp`, `device/cpu/device_impl.cpp`, `device/device.cpp`, `bvh/sort.cpp`, `bvh/build.h`, `bvh/octree.h`

## 实现细节

1. **TBB 集成**：`TaskPool` 直接将任务交由 `tbb::task_group` 调度，利用 TBB 的工作窃取算法实现高效并行。取消操作先调用 `tbb_group.cancel()` 再 `wait()`。

2. **引用计数调度器**：`TaskScheduler` 使用引用计数允许多个 Cycles 实例共享同一 TBB 线程池。仅首个调用 `init()` 的实例会真正配置线程数。

3. **全局线程控制**：当 `TBB_INTERFACE_VERSION_MAJOR >= 10` 时，使用 `tbb::global_control` 限制 TBB 的最大并行度。

4. **专用线程池设计**：`DedicatedTaskPool` 的工作线程在 `thread_wait_pop` 中阻塞等待新任务。`cancel()` 通过清空队列并更新计数实现快速取消，而非终止线程。析构时设置 `do_exit` 标志并通知工作线程退出。

5. **统计收集**：`TaskPool` 在创建时记录时间戳，`wait_work()` 完成时可计算总耗时和任务数并填入 `Summary`。

## 关联文件

- `util/thread.h` - 提供底层线程和同步原语
- `util/tbb.h` - TBB 并行框架头文件封装
- `util/time.h` - 时间函数（`time_dt()`）
