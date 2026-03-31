# guarded_allocator.h / guarded_allocator.cpp - 受保护的内存分配器

## 概述
本文件实现了 Cycles 渲染器的受保护内存分配器 `GuardedAllocator`，这是一个符合 STL 分配器接口的模板类，用于追踪所有通过 STL 容器（如 `vector`）进行的内存分配和释放。支持两种后端：Blender 的 `MEM_guardedalloc`（嵌入 Blender 时）和标准 `malloc/free`（独立运行时）。通过全局 `Stats` 对象记录内存使用量和峰值。

## 类与结构体

### `template<typename T> class GuardedAllocator`
符合 C++ 分配器要求的模板类。

#### 核心方法
| 方法 | 说明 |
|------|------|
| `T *allocate(size_t n, const void *hint)` | 分配 `n*sizeof(T)` 字节内存，并记录到统计系统 |
| `void deallocate(T *p, size_t n)` | 释放内存并更新统计 |
| `T *address(T &x)` / `const T *address(const T &x)` | 取地址 |
| `size_t max_size()` | 返回最大可分配大小 |

#### 分配后端
- `WITH_BLENDER_GUARDEDALLOC`：使用 `MEM_mallocN_aligned(size, 16, "Cycles Alloc")`
- 否则：使用标准 `malloc` / `free`

#### MSVC 特殊处理
在 MSVC 下，对 `std::_Container_proxy` 类型进行特殊化 rebind，使用标准分配器而非 GuardedAllocator，以避免：
- 静态对象初始化顺序问题（全局 `Stats` 可能未初始化）
- 运行时切换分配器类型的兼容性问题

### `MEM_GUARDED_CALL` 宏
安全调用函数的宏，捕获 `std::bad_alloc` 异常并设置 Progress 错误状态：
```cpp
MEM_GUARDED_CALL(progress, func, args...)
```

## 核心函数

### 内部 API
| 函数 | 说明 |
|------|------|
| `void util_guarded_mem_alloc(size_t n)` | 记录内存分配到全局 `Stats` |
| `void util_guarded_mem_free(size_t n)` | 记录内存释放到全局 `Stats` |

### 公开查询 API
| 函数 | 说明 |
|------|------|
| `size_t util_guarded_get_mem_used()` | 获取当前内存使用量 |
| `size_t util_guarded_get_mem_peak()` | 获取内存使用峰值 |

### 实现（cpp 文件）
全局静态 `Stats` 对象使用 `static_init` 构造方式，避免零初始化覆盖问题：
```cpp
static Stats global_stats(Stats::static_init);
```

## 依赖关系
- **内部头文件**: `util/stats.h`（cpp）
- **外部依赖**: `MEM_guardedalloc.h`（Blender 嵌入时）
- **被引用**: `util/vector.h`（`vector` 类型使用 `GuardedAllocator`）, `util/aligned_malloc.cpp`

## 关联文件
- `util/stats.h` - `Stats` 内存统计类
- `util/vector.h` - 使用 `GuardedAllocator` 的向量容器
- `util/aligned_malloc.h` - 对齐内存分配（也使用 guarded 统计）
