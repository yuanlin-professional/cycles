# stats.h - 内存使用统计

## 概述
本文件实现了一个轻量级的内存使用统计类 `Stats`，用于追踪 Cycles 渲染器运行时的内存分配总量和峰值。使用原子操作保证线程安全，可在并发环境下准确统计。主要供 `GuardedAllocator` 和内存管理模块使用。

## 类与结构体

### `class Stats`
内存统计追踪类。

#### 构造方式
| 构造函数 | 说明 |
|------|------|
| `Stats()` | 默认构造，`mem_used` 和 `mem_peak` 初始化为 0 |
| `Stats(static_init_t)` | 静态初始化构造（不初始化成员），用于全局静态变量避免初始化顺序问题 |

#### 方法
| 方法 | 说明 |
|------|------|
| `void mem_alloc(size_t size)` | 记录内存分配，原子增加 `mem_used`，更新 `mem_peak` |
| `void mem_free(size_t size)` | 记录内存释放，原子减少 `mem_used`，含断言检查 |

#### 成员变量
| 成员 | 说明 |
|------|------|
| `size_t mem_used` | 当前内存使用量（字节） |
| `size_t mem_peak` | 内存使用峰值（字节） |

#### 实现细节
- `mem_alloc` 使用 `atomic_add_and_fetch_z` 原子加法
- `mem_peak` 使用 `atomic_fetch_and_update_max_z` 原子取最大值
- `mem_free` 使用 `atomic_sub_and_fetch_z` 原子减法
- `static_init` 枚举用于标识静态对象构造，避免零初始化覆盖

## 核心函数
所有功能封装在 `Stats` 类中，无独立函数。

## 依赖关系
- **内部头文件**: `util/atomic.h`
- **被引用**: `util/guarded_allocator.cpp`（全局 `global_stats` 实例）

## 关联文件
- `util/atomic.h` - 原子操作原语
- `util/guarded_allocator.h` - 受保护的内存分配器（主要消费者）
