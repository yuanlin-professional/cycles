# stack_allocator.h - 栈上内存分配器

## 概述

`stack_allocator.h` 实现了一个可与 STL 容器配合使用的栈分配器 `StackAllocator`。该分配器优先使用栈上的固定大小缓冲区进行内存分配，仅在栈空间不足时回退到堆内存分配。这对于小规模、生命周期短的临时容器特别有用，可以避免频繁的堆分配开销。栈缓冲区以 16 字节对齐。

## 类与结构体

### `template<int SIZE, typename T> class StackAllocator`

符合 STL 分配器要求的栈分配器模板类。模板参数 `SIZE` 指定栈缓冲区可容纳的最大元素数，`T` 为元素类型。类使用 `ccl_try_align(16)` 修饰以尝试 16 字节对齐。

**类型定义（STL Allocator 必需）：**
- `size_type` = `size_t`
- `difference_type` = `ptrdiff_t`
- `pointer` = `T *`
- `const_pointer` = `const T *`
- `reference` = `T &`
- `const_reference` = `const T &`
- `value_type` = `T`

**成员变量（private）：**
- `int pointer_` -- 栈缓冲区中的当前分配位置（偏移量）
- `bool use_stack_` -- 是否使用栈分配（跨类型 rebind 时设为 false）
- `T data_[SIZE]` -- 栈上的固定大小数据缓冲区

**构造函数：**
- `StackAllocator()` -- 默认构造，启用栈分配
- `StackAllocator(const StackAllocator &)` -- 拷贝构造，重置指针但保持栈分配
- `template<class U> StackAllocator(const StackAllocator<SIZE, U> &)` -- 跨类型构造，禁用栈分配（因为数据类型不同）

**内存分配/释放：**
- `T *allocate(const size_t n, const void *hint = nullptr)` -- 分配 n 个元素的内存。如果栈空间足够且允许栈分配，则从栈缓冲区分配；否则回退到堆分配（支持 `WITH_BLENDER_GUARDEDALLOC` 配置）
- `void deallocate(T *p, const size_t n)` -- 释放内存。判断指针是否在栈缓冲区范围内：若在堆上则释放，若在栈上则不做操作（栈内存由对象生命周期管理）

**对象构造/析构：**
- `void construct(T *p, const T &val)` -- 在给定地址上使用 placement new 构造对象
- `void destroy(T *p)` -- 调用对象的析构函数

**其他方法：**
- `T *address(T &x) const` / `const T *address(const T &x) const` -- 返回引用的地址
- `size_t max_size() const` -- 返回最大分配大小（`size_t(-1)`）

**rebind 机制：**
- `template<class U> struct rebind` -- 允许将分配器重新绑定到其他类型 `U`

## 核心函数

该文件中所有功能均封装在 `StackAllocator` 类内部，无独立函数。

## 依赖关系

- **内部头文件**: `util/defines.h`, `util/guarded_allocator.h`
- **标准库**: `<cstddef>`
- **被引用**: `bvh/build.cpp`

## 关联文件

- `util/guarded_allocator.h` -- 提供堆内存分配时的内存追踪功能（`util_guarded_mem_alloc` / `util_guarded_mem_free`）
- `util/vector.h` -- `vector` 使用 `GuardedAllocator` 作为默认分配器，`StackAllocator` 可替换它用于临时容器
- `bvh/build.cpp` -- 在 BVH 构建过程中使用栈分配器优化临时数据的分配
