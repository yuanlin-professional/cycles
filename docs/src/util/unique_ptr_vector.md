# unique_ptr_vector.h - 独占指针向量容器

## 概述

`unique_ptr_vector.h` 实现了一个封装 `vector<unique_ptr<T>>` 的便捷容器类 `unique_ptr_vector<T>`。该容器的核心设计理念是：索引和迭代器直接返回裸指针值（而非 `unique_ptr` 引用），简化了日常使用。同时提供了基于 swap 的快速删除方法和批量删除功能，专为场景节点管理等场景优化。

## 类与结构体

### `template<typename T> class unique_ptr_vector`

管理独占所有权指针集合的容器类。

**成员变量（protected）：**
- `vector<unique_ptr<T>> data` -- 底层存储

**基本操作：**
- `T *operator[](const size_t i) const` -- 下标访问，直接返回裸指针
- `unique_ptr<T> steal(const size_t i)` -- 窃取第 i 个元素的所有权，原位置变为空
- `void push_back(unique_ptr<T> &&value)` -- 追加元素（右值引用，转移所有权）
- `bool empty() const` -- 是否为空
- `size_t size() const` -- 元素数量
- `void clear()` -- 清空所有元素（析构所有持有的对象）
- `void free_memory()` -- 彻底释放内存

**删除操作：**
- `void erase(const T *value)` -- 按指针值删除元素，保持顺序（线性查找，O(n) 移动）
- `void erase_by_swap(const T *value)` -- 将目标元素与末尾交换后删除，不保持顺序但更快
- `void erase_in_set(const set<T *> &values)` -- 批量删除指针集合中的所有匹配元素，使用交换到末尾策略

**排序：**
- `template<typename Compare> void stable_sort(Compare compare)` -- 稳定排序，比较器接受裸指针参数，内部自动适配 `unique_ptr` 的比较

**类型转换：**
- `operator const vector<T *> &()` -- 隐式转换为只读的裸指针 vector 引用（前提：`sizeof(unique_ptr<T>) == sizeof(T *)`，即零开销假设），通过 `reinterpret_cast` 实现

### `struct ConstIterator`

常量迭代器，解引用返回 `const T *`（裸指针），支持随机访问迭代器的全部操作（`++`、`+`、`+=`、`-` 差值、`==`/`!=` 比较）。

### `struct Iterator`

可变迭代器，解引用返回 `T *`（裸指针），接口同 `ConstIterator`。

## 核心函数

该文件中所有功能均封装在 `unique_ptr_vector` 类及其内部迭代器中，无独立函数。

## 依赖关系

- **内部头文件**: `util/algorithm.h`, `util/set.h`, `util/unique_ptr.h`, `util/vector.h`
- **标准库**: `<cassert>`
- **被引用**: `scene/scene.h`, `scene/shader_graph.h`, `scene/pass.h`, `scene/procedural.h`

## 关联文件

- `util/unique_ptr.h` -- 提供 `unique_ptr` 和 `make_unique` 的命名空间别名
- `util/vector.h` -- 底层存储容器
- `util/set.h` -- `erase_in_set` 方法使用 `set<T *>` 作为参数
- `util/algorithm.h` -- 提供 `std::stable_sort` 和 `std::swap`
- `scene/scene.h` -- 使用 `unique_ptr_vector` 管理场景中的对象、光源等节点
- `scene/shader_graph.h` -- 使用 `unique_ptr_vector` 管理着色器图中的节点
