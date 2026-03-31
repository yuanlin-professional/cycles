# vector.h - 带内存追踪的 std::vector 子类封装

## 概述

`vector.h` 定义了 Cycles 自有的 `vector<T>` 模板类，继承自 `std::vector`。子类化的目的有两个：一是使用自定义的 `GuardedAllocator` 追踪内存使用量和峰值；二是提供 `free_memory()` 方法确保容量真正归零释放。该类是 Cycles 中最基础、使用最广泛的容器类型。

## 类与结构体

### `template<typename value_type, typename allocator_type = GuardedAllocator<value_type>> class vector`

继承自 `std::vector<value_type, allocator_type>`。

**类型别名：**
- `BaseClass` = `std::vector<value_type, allocator_type>`

**构造函数：**
- 继承 `std::vector` 的所有构造函数（`using BaseClass::vector`）

**方法：**
- `void free_memory()` -- 通过与空 vector 交换（swap trick）彻底释放所有内存，包括 capacity 占用的空间
- `operator std::vector<value_type>()` -- 隐式转换为标准 `std::vector`，用于与外部 API 交互

## 核心函数

该文件中所有功能均封装在 `vector` 类内部，无独立函数。

## 依赖关系

- **内部头文件**: `util/guarded_allocator.h`
- **标准库**: `<cstring>`, `<vector>`
- **被引用**: 该文件是 Cycles 中引用最广泛的工具头文件之一，被 49 个以上文件引用，包括 `util/array.h`, `util/unique_ptr_vector.h`, `util/string.h`, `util/path.h`, `scene/` 目录下大量文件, `device/` 目录下大量文件, `bvh/` 目录下多个文件, `integrator/` 目录下多个文件等

## 关联文件

- `util/guarded_allocator.h` -- 提供内存追踪分配器，记录 Cycles 的内存使用峰值
- `util/array.h` -- 高性能数组容器，可与 `vector` 互相赋值
- `util/unique_ptr_vector.h` -- 内部使用 `vector<unique_ptr<T>>` 存储数据
