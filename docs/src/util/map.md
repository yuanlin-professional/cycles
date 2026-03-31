# map.h - 关联容器类型别名与内存释放工具

## 概述

`map.h` 将标准库中的 `std::map`、`std::pair`、`std::unordered_map` 和 `std::unordered_multimap` 引入到 Cycles 命名空间 (`CCL_NAMESPACE`) 中，使得项目代码无需 `std::` 前缀即可使用这些类型。同时提供了一个模板辅助函数 `map_free_memory` 用于彻底释放关联容器的内部内存。

## 类与结构体

该文件不定义新的类或结构体，仅通过 `using` 声明引入标准库类型：

- `map` -- `std::map`（有序关联容器）
- `pair` -- `std::pair`（键值对）
- `unordered_map` -- `std::unordered_map`（哈希关联容器）
- `unordered_multimap` -- `std::unordered_multimap`（允许重复键的哈希关联容器）

## 核心函数

### `template<typename T> static void map_free_memory(T &data)`

使用 swap 技巧彻底释放关联容器的所有内部内存。创建一个空容器并与目标容器交换，使得原容器的内存被空容器的析构函数释放。此方法适用于所有支持 `swap()` 的容器类型。

**参数：**
- `data` -- 待释放内存的容器引用

## 依赖关系

- **内部头文件**: 无
- **标准库**: `<map>`, `<unordered_map>`
- **被引用**: `util/path.cpp`, `session/merge.cpp`, `session/denoising.cpp`, `scene/shader_graph.h`, `scene/shader.h`, `scene/object.cpp`, `scene/colorspace.cpp`, `device/queue.h`, `device/multi/device.cpp`, `graph/node_enum.h`, `graph/node_type.h`

## 关联文件

- `util/set.h` -- 类似的集合容器类型别名封装
- `util/vector.h` -- 类似地提供了 `free_memory()` 方法
