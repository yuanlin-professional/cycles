# algorithm.h - 标准算法函数别名封装

## 概述

`algorithm.h` 将标准库 `<algorithm>` 中常用的算法函数引入到 Cycles 命名空间中，使项目代码可以直接使用 `sort`、`swap` 等函数名而无需 `std::` 前缀。该文件使用 `// IWYU pragma: export` 注释，指示 Include-What-You-Use 工具将 `<algorithm>` 视为导出头文件，即包含 `util/algorithm.h` 的文件自动获得 `<algorithm>` 的所有声明。

## 类与结构体

该文件不定义新的类或结构体。

## 核心函数

通过 `using` 声明引入以下标准算法函数：

- `remove` -- `std::remove`，移除容器中等于指定值的元素（逻辑移除，需配合 `erase` 使用）
- `sort` -- `std::sort`，快速排序（不保证稳定性，平均 O(n log n)）
- `stable_sort` -- `std::stable_sort`，稳定排序，保持相等元素的相对顺序
- `swap` -- `std::swap`，交换两个值
- `upper_bound` -- `std::upper_bound`，在有序范围中查找第一个大于指定值的元素（二分查找，O(log n)）

## 依赖关系

- **内部头文件**: 无
- **标准库**: `<algorithm>`（通过 IWYU pragma 导出）
- **被引用**: `util/unique_ptr_vector.h`, `util/path.cpp`, `util/math_cdf.cpp`, `subd/split.cpp`, `scene/stats.cpp`, `scene/shader_graph.cpp`, `scene/mesh_subdivision.cpp`, `device/queue.cpp`, `bvh/sort.cpp`, `bvh/split.cpp`, `bvh/build.cpp`, `bvh/binning.cpp`

## 关联文件

- `util/vector.h` -- 算法函数通常作用于 `vector` 容器
- `util/unique_ptr_vector.h` -- 内部使用 `std::stable_sort` 进行排序
