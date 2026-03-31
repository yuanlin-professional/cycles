# semaphore.h - 计数信号量

## 概述

`semaphore.h` 实现了一个计数信号量 `thread_counting_semaphore`，用于限制对共享资源的并发访问线程数。功能类似于 C++20 的 `std::counting_semaphore`，但基于 Cycles 自身的互斥锁和条件变量原语实现，以保持对 C++17 及更早标准的兼容性。

## 类与结构体

### `thread_counting_semaphore`

基于互斥锁和条件变量的计数信号量实现。

| 成员/方法 | 说明 |
|-----------|------|
| `thread_counting_semaphore(int count)` | 构造函数，`count` 为允许的最大并发数 |
| `acquire()` | 获取信号量（计数 -1），若计数为 0 则阻塞等待 |
| `release()` | 释放信号量（计数 +1），唤醒一个等待线程 |

**不可复制**：拷贝构造函数被显式删除。

**保护成员：**

| 成员 | 说明 |
|------|------|
| `thread_mutex mutex` | 保护计数器的互斥锁 |
| `thread_condition_variable condition` | 用于阻塞/唤醒的条件变量 |
| `int count` | 当前可用计数 |

## 核心函数/宏定义

无额外宏定义。所有功能通过类方法提供。

## 依赖关系

- **内部头文件**: `util/thread.h`（提供 `thread_mutex`, `thread_scoped_lock`, `thread_condition_variable`）
- **被引用**: 当前代码库中无直接引用（可作为通用并发控制工具使用）

## 实现细节

1. **经典信号量模式**：`acquire()` 在 `while(count == 0)` 循环中等待条件变量，使用 while 而非 if 以防止虚假唤醒 (spurious wakeup)。

2. **粒度控制**：`release()` 使用 `notify_one()` 而非 `notify_all()`，每次仅唤醒一个等待线程，避免惊群效应。

3. **RAII 安全**：`acquire()` 中使用 `thread_scoped_lock` 管理锁的生命周期，`wait()` 调用会在阻塞期间自动释放锁，被唤醒后重新获取。

## 关联文件

- `util/thread.h` - 提供互斥锁和条件变量类型定义
