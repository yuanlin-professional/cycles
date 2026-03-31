# set.h - 集合容器类型别名封装

## 概述

`set.h` 将标准库中的 `std::set` 和 `std::unordered_set` 引入到 Cycles 命名空间中，使项目代码可以直接使用 `set` 和 `unordered_set` 而无需 `std::` 前缀。在 MSVC 2015 及以上版本中，额外包含 `<iterator>` 头文件以确保兼容性。

## 类与结构体

该文件不定义新的类或结构体，仅通过 `using` 声明引入标准库类型：

- `set` -- `std::set`（有序集合容器）
- `unordered_set` -- `std::unordered_set`（哈希集合容器）

## 核心函数

无。该文件仅包含类型别名声明。

## 依赖关系

- **内部头文件**: 无
- **标准库**: `<set>`, `<unordered_set>`；在 MSVC (`_MSC_VER >= 1900`) 下额外包含 `<iterator>`
- **被引用**: `util/unique_ptr_vector.h`, `util/path.cpp`, `subd/split.h`, `scene/shader_graph.h`, `scene/osl.h`, `scene/mesh.cpp`, `scene/mesh.h`, `scene/object.cpp`, `scene/geometry.h`, `scene/attribute.h`, `device/metal/device.mm`

## 关联文件

- `util/map.h` -- 类似的关联容器类型别名封装
- `util/concurrent_set.h` -- 基于 TBB 的线程安全集合容器
