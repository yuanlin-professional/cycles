# progress.h - 渲染进度追踪与状态管理

## 概述
本文件实现了 `Progress` 类，用于在 Cycles 渲染过程中跟踪和通信进度状态、时间信息、取消/错误信号以及更新回调。该类设计为线程安全（除构造/析构外），支持从工作线程向主线程报告渲染进度。通过像素采样计数和可选的时间限制两种方式计算进度百分比。

## 类与结构体

### `class Progress`
渲染进度管理的核心类。

#### 取消与错误管理
| 方法 | 说明 |
|------|------|
| `void set_cancel(const string &msg)` | 设置取消标志和消息 |
| `bool get_cancel() const` | 查询取消状态（同时触发取消回调） |
| `string get_cancel_message() const` | 获取取消原因消息 |
| `void set_cancel_callback(function<void()>)` | 设置取消检查回调 |
| `void set_error(const string &msg)` | 设置错误（同时触发取消） |
| `bool get_error() const` | 查询错误状态 |
| `string get_error_message() const` | 获取错误消息 |

#### 时间与采样统计
| 方法 | 说明 |
|------|------|
| `void set_start_time()` | 记录开始时间 |
| `void set_render_start_time()` | 记录渲染开始时间（排除场景同步） |
| `void set_time_limit(double)` | 设置渲染时间限制 |
| `void get_time(double &total, double &render)` | 获取总耗时和渲染耗时 |
| `void set_total_pixel_samples(uint64_t)` | 设置总像素采样数 |
| `void add_samples(uint64_t pixel_samples, int tile_sample)` | 增加已完成采样数 |
| `double get_progress() const` | 获取进度百分比 [0.0, 1.0]，取采样进度和时间进度的较大值 |
| `int get_current_sample() const` | 获取当前 tile 采样数 |
| `void add_finished_tile(bool denoised)` | 记录完成的 tile（区分渲染/降噪） |

#### 状态消息
| 方法 | 说明 |
|------|------|
| `void set_status(const string &status, const string &substatus)` | 设置主状态和子状态消息 |
| `void set_substatus(const string &substatus)` | 仅设置子状态消息 |
| `void set_sync_status(...)` / `set_sync_substatus(...)` | 设置场景同步状态（优先显示） |
| `void get_status(string &, string &)` | 获取当前状态（同步状态优先于渲染状态） |

#### 回调
| 方法 | 说明 |
|------|------|
| `void set_update()` | 触发更新回调 |
| `void set_update_callback(function<void()>)` | 设置状态更新回调函数 |

#### 保护成员
| 成员 | 说明 |
|------|------|
| `thread_mutex progress_mutex` / `update_mutex` | 线程互斥锁 |
| `uint64_t pixel_samples` / `total_pixel_samples` | 像素采样计数（所有像素的总采样数） |
| `int current_tile_sample` | 最后更新的 tile 的当前采样数 |
| `int rendered_tiles` / `denoised_tiles` | 已完成的渲染/降噪 tile 计数 |
| `double start_time` / `render_start_time` / `time_limit` / `end_time` | 时间追踪 |
| `volatile bool cancel` / `error` | 取消/错误状态标志 |

## 核心函数
所有功能封装在 `Progress` 类的方法中，无独立函数。

## 依赖关系
- **内部头文件**: `util/string.h`, `util/thread.h`, `util/time.h`
- **被引用**: `session/session.h`, `scene/scene.h`, `render/` 下的渲染管线模块

## 关联文件
- `util/thread.h` - `thread_mutex`, `thread_scoped_lock` 线程同步原语
- `util/time.h` - `time_dt()`, `scoped_timer` 时间工具
- `session/session.h` - 渲染会话持有 `Progress` 实例
