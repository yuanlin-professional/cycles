# array.h - 高性能对齐内存数组容器

## 概述

`array.h` 实现了一个轻量级的模板数组类 `array<T>`，作为 `std::vector` 的简化替代方案。该容器专为渲染性能优化设计：不在 resize/alloc 时清零内存，不调用元素的构造函数和析构函数，并使用对齐内存分配以适配 CPU 原生数据类型（默认对齐到 `MIN_ALIGNMENT_CPU_DATA_TYPES`）。在性能分析（profiling）中，标准容器的初始化开销曾显著影响渲染效率，该类正是为解决此问题而设计。

## 类与结构体

### `template<typename T, const size_t alignment = MIN_ALIGNMENT_CPU_DATA_TYPES> class array`

核心模板数组类，模板参数 `T` 为元素类型，`alignment` 为内存对齐字节数。

**成员变量（protected）：**
- `T *data_` -- 指向已分配内存的指针
- `size_t datasize_` -- 当前存储的元素数量
- `size_t capacity_` -- 已分配的总容量（元素数）

**构造与赋值：**
- `array()` -- 默认构造，数据指针为 nullptr，大小和容量为 0
- `explicit array(const size_t newsize)` -- 分配指定大小的未初始化内存
- `array(const array &from)` -- 拷贝构造，使用 `memcpy` 复制数据
- `array(array &&from)` -- 移动构造，窃取源对象的数据指针
- `operator=(const array &from)` -- 拷贝赋值
- `operator=(const vector<T> &from)` -- 从 `vector<T>` 赋值

**核心方法：**
- `T *resize(const size_t newsize)` -- 调整大小，仅在新容量大于当前容量时重新分配；分配失败返回 nullptr
- `T *resize(const size_t newsize, const T &value)` -- 调整大小并用指定值填充新增元素
- `void clear()` -- 释放内存并重置所有状态
- `void reserve(const size_t newcapacity)` -- 预分配容量，不改变 datasize_
- `void push_back_slow(const T &t)` -- 追加元素（非性能关键路径使用），容量不足时按 1.2 倍增长
- `void push_back_reserved(const T &t)` -- 追加元素（已确保有足够预分配容量）
- `void append(const array<T> &from)` -- 批量追加另一个数组的所有元素

**数据操作：**
- `void steal_data(array &from)` -- 窃取另一数组的数据所有权
- `void set_data(T *ptr_, size_t datasize)` -- 直接设置外部数据指针
- `T *steal_pointer()` -- 取出数据指针并放弃所有权

**访问与查询：**
- `T *data()` / `const T *data() const` -- 获取数据指针
- `T &operator[](size_t i) const` -- 下标访问（带 assert 边界检查）
- `size_t size() const` -- 返回元素数量
- `size_t capacity() const` -- 返回已分配容量
- `size_t empty() const` -- 是否为空
- `T *begin()` / `T *end()` -- 迭代器支持

**内部内存管理（protected）：**
- `T *mem_allocate(const size_t N)` -- 使用 `util_aligned_malloc` 对齐分配
- `void mem_free(T *mem, const size_t N)` -- 使用 `util_aligned_free` 释放
- `void mem_copy(T *mem_to, const T *mem_from, const size_t N)` -- 使用 `memcpy` 复制

## 核心函数

该文件中所有功能均封装在 `array` 类内部，无独立函数。

## 依赖关系

- **内部头文件**: `util/aligned_malloc.h`, `util/vector.h`
- **标准库**: `<cassert>`, `<cstring>`
- **被引用**: `util/disjoint_set.h`, `scene/shader_nodes.h`, `scene/svm.h`, `scene/osl.h`, `scene/particles.h`, `scene/mesh.h`, `scene/object.h`, `scene/camera.h`, `scene/curves.h`, `graph/node_type.h`, `graph/node.h`, `device/memory.h`, `bvh/build.h`, `bvh/bvh.h`, `session/merge.cpp`, `integrator/denoiser_oidn.cpp`, `app/cycles_precompute.cpp` 等

## 关联文件

- `util/vector.h` -- `array` 的赋值运算符支持从 `vector<T>` 赋值
- `util/aligned_malloc.h` -- 提供对齐内存分配/释放的底层实现
- `device/memory.h` -- 设备内存管理中大量使用 `array` 存储数据
